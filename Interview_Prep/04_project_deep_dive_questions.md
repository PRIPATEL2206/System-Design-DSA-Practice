# Project Deep-Dive Questions — Based on YOUR Resume

Interviewers will pick 1-2 projects from your resume and go deep. Prepare answers for all of these.

---

## Project 1: Large-Scale ETL Pipeline (TCS — 125M+ HCP Records)

### Questions They Will Ask:

**Q1: Walk me through the architecture of your ETL pipeline.**
> Expected: Draw the data flow from source to sink, explain each AWS service's role.

**Answer Framework:**
```
Sources → Ingestion → Processing → Validation → Loading → Serving

- Sources: Aurora (CDC via DMS), S3 (file drops from partners), external APIs
- Ingestion: S3 as landing zone, Glue Crawlers for schema discovery
- Processing: Glue ETL (PySpark) for transformation, deduplication
- Validation: Data quality checks between zones (null checks, referential integrity)
- Loading: Redshift (COPY command from S3, using manifests)
- Serving: Redshift Spectrum for ad-hoc, materialized views for dashboards
- Orchestration: Step Functions to coordinate multi-step pipelines
- Monitoring: CloudWatch alarms + SNS notifications
```

**Q2: How did you handle data quality for 125M records?**
> Show systematic approach, not ad-hoc.

**Answer Points:**
- Schema validation at ingestion (reject malformed records to DLQ)
- Null/duplicate checks in PySpark transformations
- Referential integrity checks (e.g., every HCP record has valid region)
- Row count reconciliation: source count vs loaded count
- Automated alerts when quality drops below threshold
- "Quarantine" zone for records that fail validation — manual review

**Q3: You mention 10% AWS cost reduction. How did you achieve that?**
> Be specific with numbers and techniques.

**Answer Points:**
- Identified over-provisioned Glue jobs: reduced DPUs from 10 to 6 (jobs still completed within SLA)
- Moved infrequently accessed S3 data to Intelligent Tiering (30% of data was rarely queried)
- Reserved capacity for Redshift vs on-demand
- Consolidated multiple small Lambda functions into single batched invocations
- Used Glue job bookmarks to avoid reprocessing (reduced redundant compute by ~20%)
- S3 lifecycle policies: raw data → IA after 30 days → Glacier after 90 days
- Total savings: ~₹1 lakh/month

**Q4: How do you handle failures in the pipeline?**
```
Strategy:
1. Idempotent jobs: Re-running doesn't create duplicates (upsert logic)
2. Job bookmarks: Track what's been processed
3. Dead Letter Queue: Failed records go to SQS DLQ for manual triage
4. Retry with backoff: Transient failures auto-retry (max 3 attempts)
5. Alerting: CloudWatch → SNS → Slack within 5 minutes of failure
6. Manual intervention: Run-book for common failures (schema change, source down)
```

**Q5: What's the most challenging bug you faced in this pipeline?**
> Prepare a specific incident with STAR format.

**Framework:**
- **Situation:** Data duplication in Redshift after a Glue job timeout/restart
- **Task:** Identify root cause and prevent future occurrences
- **Action:** Added deduplication step using window functions (ROW_NUMBER by PK, keep latest), implemented job bookmarks properly, added reconciliation counts
- **Result:** Zero duplicate incidents since fix, added monitoring for row count anomalies

**Q6: How would you scale this to 1B records/day?**
```
Changes needed:
- Partition processing: Process by region/date partitions in parallel
- Use EMR over Glue for heavier workloads (more control over cluster sizing)
- Introduce streaming layer: Kinesis for real-time + batch backfill
- Separate hot/cold storage paths
- Add data lake (S3 + Iceberg) for ACID transactions at scale
- Federated queries instead of loading everything into Redshift
```

---

## Project 2: Scalable Real-Time Chat App (FastAPI + WebSocket)

### Questions They Will Ask:

**Q1: Why did you choose WebSockets over other real-time technologies?**
```
Trade-off analysis:
- Polling: Simple but wasteful (N users × polling interval = too many requests)
- Long polling: Better but still HTTP overhead per message
- SSE: One-way only (server → client), not suitable for chat
- WebSocket: Full-duplex, persistent connection, low latency ✓

For chat: WebSocket is the right choice because:
- Bidirectional communication needed
- Low latency requirements (<100ms)
- Persistent connection reduces overhead vs HTTP per message
```

**Q2: How do you scale WebSocket connections across multiple servers?**
```
Challenge: WebSocket connections are stateful (tied to one server)

Solutions:
1. Sticky sessions (simple): Load balancer routes user to same server
2. Redis Pub/Sub (recommended):
   - Each server subscribes to Redis channels
   - Message sent → publish to Redis → all servers relay to their connected clients
3. Shared state: Connection registry in Redis (user_id → server_id)

My implementation approach:
- Redis pub/sub for message fan-out
- Connection manager per server instance
- Heartbeat mechanism to detect dead connections
```

**Q3: How do you handle connection drops and message delivery?**
```
Mechanisms:
1. Heartbeat ping/pong every 30 seconds (detect dead connections)
2. Client-side reconnection with exponential backoff
3. Message persistence in DB (PostgreSQL/Cassandra)
4. On reconnect: fetch missed messages since last_seen_timestamp
5. Message acknowledgment: client sends ACK, server retries if no ACK
6. Offline queue: Messages queued in Redis, delivered on reconnect
```

**Q4: What's your message storage strategy?**
```
Design decisions:
- Recent messages: Redis (last 50 per room for fast access)
- Full history: PostgreSQL with partitioning by room + date
- Media/files: S3 with pre-signed URLs
- Search: Elasticsearch index on message content (if needed)

Schema:
messages(id, room_id, sender_id, content, type, created_at, delivered_at, read_at)
Index on: (room_id, created_at) for pagination
```

**Q5: How would you add end-to-end encryption?**
```
Approach: Signal Protocol (simplified)
- Each user generates public/private key pair
- Public keys exchanged via server during room join
- Messages encrypted client-side before sending over WebSocket
- Server stores only encrypted blobs (can't read content)
- Key rotation on member add/remove from group
```

---

## Project 3: Plant Disease Classification (CNN + TensorFlow)

### Questions They Will Ask:

**Q1: Explain your model architecture and why CNN?**
```
Why CNN: Image classification is CNN's strength
- Spatial feature extraction (leaves have texture patterns)
- Translation invariance (disease can be anywhere on leaf)
- Hierarchical features (edges → textures → patterns → disease)

Architecture:
- Base: Transfer learning with MobileNetV2 (pre-trained on ImageNet)
- Custom head: GlobalAveragePooling → Dense(256) → Dropout(0.3) → Dense(num_classes)
- Input: 224x224x3 images
- Output: Softmax over disease classes

Why MobileNetV2: Lightweight, deployable on edge devices, good accuracy/size ratio
```

**Q2: How did you handle overfitting with limited data?**
```
Techniques used:
1. Data Augmentation:
   - Random rotation (±30°), horizontal flip, zoom (0.8-1.2x)
   - Color jitter (brightness, contrast) to simulate lighting conditions
   - Random crop and resize
2. Transfer Learning: Pre-trained weights provide generalization
3. Dropout: 0.3 in classification head
4. Early Stopping: Monitor val_loss, patience=5
5. Regularization: L2 on dense layers
6. Stratified split: Equal class distribution in train/val/test
```

**Q3: How would you deploy this model in production?**
```
Architecture:
[Mobile App / Web UI] → [FastAPI Backend] → [TensorFlow Serving / ONNX Runtime]
                                                    ↓
                                           [S3 Model Registry]
                                           [CloudWatch Monitoring]

Steps:
1. Model export: SavedModel format or ONNX conversion
2. Serving: TensorFlow Serving in Docker container
3. API: FastAPI endpoint for image upload + prediction
4. Preprocessing: Resize + normalize in API layer
5. Monitoring: Log predictions, track accuracy drift
6. Retraining: Trigger when accuracy drops below threshold
7. A/B testing: Shadow new model alongside current one
```

**Q4: What metrics did you use and what accuracy did you achieve?**
```
Metrics:
- Accuracy: ~95% on test set
- Precision/Recall per class (important because misclassifying disease is costly)
- Confusion matrix to identify commonly confused diseases
- F1-score (harmonic mean, handles class imbalance)

Why not just accuracy:
- Imbalanced classes (some diseases are rare)
- False negative cost is high (missing a disease = crop loss)
- Used weighted F1 to account for class imbalance
```

---

## Project 4: Trust Management System (Django + PostgreSQL)

### Questions They Will Ask:

**Q1: Explain the role-based access control (RBAC) implementation.**
```
Design:
- Roles: SuperAdmin, Admin, Manager, Member, Viewer
- Permissions: defined per resource (donations, expenses, members, reports)
- Implementation: Django's built-in auth + custom permission classes

Models:
Role → has many → Permissions
User → has one → Role
User → belongs to → Trust

Permission check flow:
1. Request comes in → JWT decoded → user_id + role extracted
2. View decorator checks: @permission_required("donations.approve")
3. Row-level: Filter querysets by user's trust_id (multi-tenant isolation)
4. Admin approval: Actions like expense > ₹10k go to approval queue
```

**Q2: How did you design the approval workflow?**
```
State machine pattern:
DRAFT → SUBMITTED → UNDER_REVIEW → APPROVED/REJECTED

Implementation:
- Approval model: (request_id, approver_id, status, comments, timestamp)
- Config: Approval rules per action (amount threshold, required approvers)
- Notifications: On state change, notify relevant users
- Audit trail: Every state transition logged with user + timestamp

Edge cases:
- Multi-level approval (amount > 1L needs 2 approvers)
- Delegation (if approver is on leave)
- Timeout (auto-escalate after 48 hours)
```

**Q3: How do you handle multi-tenancy (multiple trusts on same platform)?**
```
Approach: Shared database, schema-level isolation

- Every model has trust_id foreign key
- Custom QuerySet manager: filters by request.user.trust_id automatically
- Middleware extracts trust context from JWT
- Database indexes include trust_id for performance
- No cross-trust data leakage possible (enforced at ORM level)

Alternative considered:
- Separate schemas per trust (more isolation, harder to maintain)
- Separate databases (maximum isolation, operational overhead)
- Chose shared DB for simplicity at current scale
```

---

## General Framework for Project Questions

### Use STAR-T Method:

| Letter | What | Example |
|--------|------|---------|
| **S**ituation | Context | "We had 125M HCP records across multiple AWS services..." |
| **T**ask | Your responsibility | "I was responsible for optimizing the pipeline performance..." |
| **A**ction | What you did | "I implemented job bookmarks, partitioning, and DPU tuning..." |
| **R**esult | Measurable outcome | "Reduced processing time by 40% and costs by 10%..." |
| **T**echnical | Deep dive ready | "The key insight was using pushdown predicates in Glue..." |

### Common Follow-up Patterns:

1. "What would you do differently?" → Show growth mindset
2. "How would you scale this 10x?" → Show architectural thinking
3. "What was the hardest bug?" → Show debugging skills
4. "How did you test this?" → Show quality awareness
5. "What trade-offs did you make?" → Show decision-making maturity
