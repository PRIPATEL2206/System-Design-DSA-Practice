# System Design (HLD) Questions — Likely in Your Interview

## Based on Your Resume, These Are Most Probable

---

## Q1: Design a Large-Scale ETL Pipeline

> "Design a system that processes 125M+ healthcare records daily from multiple sources into a data warehouse."

**Why they'll ask:** It's literally your job at TCS.

### Expected Answer Structure:

```
1. Requirements Clarification (2 min)
   - Data volume: 125M records/day, ~50GB raw
   - Sources: S3, Aurora, external APIs
   - Latency SLA: T+4 hours acceptable? Or near real-time?
   - Data quality requirements

2. High-Level Architecture:
   
   [Sources] → [Ingestion Layer] → [Processing Layer] → [Storage Layer] → [Serving Layer]
   
   Sources: S3 (files), Aurora (CDC), APIs (REST pulls)
   Ingestion: AWS Glue Crawlers, Lambda triggers, SQS for buffering
   Processing: Glue ETL jobs (PySpark), Step Functions for orchestration
   Storage: S3 (raw/curated/enriched zones), Redshift (analytics)
   Serving: Redshift Spectrum, QuickSight, API layer

3. Key Design Decisions:
   - Partition strategy: date + region (for HCP data)
   - Idempotency: job bookmarks + deduplication keys
   - Error handling: Dead Letter Queue + retry with exponential backoff
   - Schema evolution: Glue Schema Registry
   - Data quality: validation step between raw → curated zones

4. Scaling & Optimization:
   - Glue auto-scaling DPUs
   - Redshift: distribution key = hcp_id, sort key = date
   - S3 lifecycle policies: raw → IA after 30 days → Glacier after 90
   - Cost: spot instances for non-critical jobs
```

### Follow-up Questions to Expect:
- How do you handle late-arriving data?
- What if a source schema changes without notice?
- How do you ensure exactly-once processing?
- How would you add real-time processing alongside batch?

---

## Q2: Design a Real-Time Chat System

> "Design WhatsApp/Slack-like real-time messaging system."

**Why they'll ask:** You built a chat app (FastAPI + WebSockets).

### Expected Answer Structure:

```
1. Requirements:
   - 1:1 and group messaging
   - Online/offline status
   - Message delivery guarantees (sent, delivered, read)
   - Media support
   - 10M DAU scale

2. Architecture:

   [Client] ←WebSocket→ [Chat Server] → [Message Queue] → [Storage]
                              ↓
                     [Presence Service]
                     [Push Notification]

3. Components:
   - Connection Manager: WebSocket server (stateful), sticky sessions
   - Message Service: Route messages, fan-out for groups
   - Storage: Messages in Cassandra (write-heavy), User data in PostgreSQL
   - Cache: Redis for recent messages + online status
   - Media: S3 + CDN for images/files
   - Push: FCM/APNs for offline users

4. Key Decisions:
   - Pull vs Push: Push via WebSocket (active), Pull for history
   - Message ordering: Timestamp + sequence number per conversation
   - Group fan-out: On-write for small groups, on-read for large channels
   - Delivery receipts: Ack-based protocol over WebSocket
```

---

## Q3: Design a Notification System

> "Design a system that sends 100M+ notifications per day across email, SMS, push."

**Why they'll ask:** Common at scale, tests queue design knowledge.

### Key Points:

```
Architecture:
[Event Sources] → [Notification Service] → [Priority Queue] → [Workers] → [Channels]
                         ↓
              [Template Engine] + [User Preferences] + [Rate Limiter]

Key Design:
- Priority queues: Critical (OTP) > High (transactions) > Low (marketing)
- Rate limiting: Per-user, per-channel limits
- Deduplication: Hash(user_id + template_id + content) within time window
- Retry: Exponential backoff with max 3 attempts
- Analytics: Track delivery, open, click rates
```

---

## Q4: Design an API Rate Limiter

> "Design a rate limiting service for a public API."

**Why they'll ask:** You mentioned cost optimization + API development.

### Key Points:

```
Algorithms:
- Token Bucket: Best for bursty traffic (recommended)
- Sliding Window Log: Most accurate but memory-heavy
- Sliding Window Counter: Good balance

Architecture:
[Client] → [API Gateway] → [Rate Limiter (Redis)] → [Backend Service]

Redis implementation:
- Key: user_id:endpoint:minute
- Value: request count
- TTL: window size
- INCR + EXPIRE atomic operation

Distributed considerations:
- Race condition: Use Redis Lua scripts for atomicity
- Multiple servers: Centralized Redis cluster
- Sticky sessions alternative: Less accurate but simpler
```

---

## Q5: Design a Data Lake Architecture

> "Design a data lake that supports both batch analytics and ML workloads."

**Why they'll ask:** Combines your AWS + ML + Data Engineering experience.

### Key Points:

```
Zones:
- Raw (Landing): Immutable, original format, partitioned by ingestion date
- Curated (Cleaned): Validated, deduplicated, standard schema
- Enriched (Feature): Aggregated, joined, ready for analytics/ML
- Sandbox: Ad-hoc exploration for data scientists

Technology:
- Storage: S3 with intelligent tiering
- Catalog: AWS Glue Data Catalog
- Processing: Glue (batch), Kinesis (stream), EMR (heavy ML)
- Query: Athena (ad-hoc), Redshift Spectrum (BI)
- ML: SageMaker reading from feature store in enriched zone

Governance:
- Lake Formation for access control
- Schema registry for evolution
- Data lineage tracking
- PII encryption + masking
```

---

## Q6: Design a Model Serving Platform

> "Design a system to serve ML models in production with A/B testing."

**Why they'll ask:** AI/ML Engineer role + your ML projects.

### Key Points:

```
Architecture:
[Client] → [API Gateway] → [Router (A/B)] → [Model Server A/B]
                                                    ↓
                                         [Feature Store (Redis)]
                                         [Model Registry (MLflow)]
                                         [Monitoring]

Components:
- Model Registry: Version control, metadata, lineage
- Feature Store: Online (Redis) for real-time, Offline (S3) for batch
- Serving: FastAPI + TensorFlow Serving / TorchServe
- A/B Router: Traffic splitting based on experiment config
- Monitoring: Prediction latency, data drift, model accuracy
- Rollback: Automated if accuracy drops below threshold

Scaling:
- Horizontal pod autoscaler based on latency
- Model caching in memory
- Batch prediction for non-real-time use cases
- Shadow mode for new models (serve but don't act)
```

---

## Q7: Design a Microservices Architecture for an Internal Portal

> "Design the backend architecture for an internal company portal with role-based access."

**Why they'll ask:** You built internal portals using Django/Flask at TCS.

### Key Points:

```
Services:
- Auth Service: JWT tokens, role management, LDAP integration
- User Service: Profiles, preferences, org hierarchy
- Portal Service: Pages, dashboards, widgets
- Notification Service: Email, in-app alerts
- Audit Service: Track all actions for compliance

Communication:
- Sync: REST between services (internal API gateway)
- Async: SQS for audit logging, notifications

Security:
- API Gateway with JWT validation
- Service mesh for internal mTLS
- Role-Based Access Control (RBAC) at gateway level
- Row-level security in database for multi-tenant data
```

---

## General Tips for Design Round

1. **Always start with requirements** — Ask clarifying questions (5 min)
2. **Estimate scale** — Users, requests/sec, storage, bandwidth
3. **Draw high-level first** — Then zoom into components
4. **Discuss trade-offs** — Don't just pick, explain WHY
5. **Relate to your experience** — "In my TCS project, we solved this by..."
6. **Address non-functional requirements** — Scalability, reliability, cost, security
7. **End with monitoring** — How do you know the system is healthy?

---

## Answer Template

```
1. CLARIFY (2 min): Functional + Non-functional requirements
2. ESTIMATE (2 min): Scale numbers (QPS, storage, bandwidth)  
3. HIGH-LEVEL DESIGN (5 min): Draw boxes + arrows, explain data flow
4. DEEP DIVE (10 min): Pick 2-3 components, explain internals
5. TRADE-OFFS (3 min): What you chose and what you gave up
6. BOTTLENECKS + SCALING (3 min): Where it breaks, how to fix
```
