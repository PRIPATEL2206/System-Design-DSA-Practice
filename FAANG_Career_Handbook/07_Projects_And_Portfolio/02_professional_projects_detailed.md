# Professional-Grade Projects — Build Your Way to Senior Engineer

> These aren't tutorials. These are production-quality systems that demonstrate
> senior engineering thinking: observability, failure handling, scalability, and trade-offs.

---

## Project 1: Real-Time Data Quality Engine

### What It Does
A service that monitors data pipelines in real-time and catches data quality issues BEFORE bad data reaches downstream consumers.

### Why It's Impressive
Every company with data pipelines has data quality problems. This solves a real $10M+ problem (bad data causes wrong business decisions). Shows you think beyond "ETL works" to "ETL works CORRECTLY."

### Architecture
```
Data Source (Kafka/Kinesis)
       │
       ▼
┌─────────────────────┐
│  Ingestion Layer    │  ← PySpark Structured Streaming
│  (stream + batch)   │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Rules Engine       │  ← Configurable quality rules (YAML/JSON)
│  - Schema check     │     - Null rate thresholds
│  - Range validation │     - Distribution drift detection
│  - Freshness check  │     - Cross-column consistency
│  - Anomaly detect   │     - Custom SQL assertions
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Alert + Action     │  ← SNS/Slack alerts
│  - Quarantine bad   │     Quarantine table
│  - Auto-fix known   │     Auto-healing rules
│  - Dashboard        │     Grafana metrics
└─────────────────────┘
```

### Tech Stack
- **Core:** Python, PySpark (Structured Streaming)
- **Rules:** YAML config files, custom DSL parser
- **Storage:** S3 (data lake), PostgreSQL (metadata/rules), Redis (real-time counters)
- **Alerting:** AWS SNS, Slack webhook
- **Dashboard:** Grafana + Prometheus metrics
- **Deployment:** Docker, AWS ECS, Terraform

### Key Features to Implement
1. **Schema Evolution Detection** — Alert when upstream schema changes unexpectedly
2. **Statistical Anomaly Detection** — Z-score and IQR on numeric columns
3. **Freshness SLA Monitoring** — Alert if data hasn't arrived within expected window
4. **Lineage Tracking** — Know which downstream tables are affected by bad upstream data
5. **Self-Healing Rules** — Auto-impute known fixable issues (e.g., timezone conversion)

### Implementation Plan (3 weeks)
```
Week 1: Core engine
- PySpark streaming job reading from Kafka/Kinesis
- Rule engine with 5 basic checks (null, range, type, uniqueness, freshness)
- YAML-based rule configuration
- Results written to PostgreSQL

Week 2: Intelligence + Alerting
- Statistical anomaly detection (rolling mean/std)
- Slack/email alerting with severity levels
- Quarantine table for flagged records
- REST API (FastAPI) for rule CRUD

Week 3: Observability + Polish
- Grafana dashboard (pass rate, alert history, latency)
- Docker Compose for local dev
- Integration tests with synthetic bad data
- README with architecture diagram
```

### Interview Talking Points
- "How do you handle late-arriving data?" → Watermarking + grace period
- "How do you avoid alert fatigue?" → Severity tiers + cooldown windows
- "How would you scale this?" → Partition rules by table/topic, horizontal workers

---

## Project 2: ML Model Serving Platform (Mini-MLflow + BentoML)

### What It Does
A platform that takes trained ML models, packages them, serves predictions via REST API, handles A/B testing between model versions, and monitors for drift.

### Why It's Impressive
MLOps is the #1 hiring gap in ML engineering. Companies have models in notebooks that never reach production. This shows you bridge that gap.

### Architecture
```
┌──────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ Model Upload │────►│  Model Registry  │────►│ Serving Engine   │
│ (CLI/API)    │     │  (versioned,     │     │ (FastAPI workers)│
│              │     │   metadata)      │     │ - A/B routing    │
└──────────────┘     └──────────────────┘     │ - Auto-scaling   │
                                              └────────┬─────────┘
                                                       │
                                              ┌────────▼─────────┐
                                              │ Monitor Service   │
                                              │ - Latency P50/99 │
                                              │ - Data drift      │
                                              │ - Prediction dist │
                                              │ - Alert on decay  │
                                              └──────────────────┘
```

### Tech Stack
- **API:** FastAPI (model serving + management API)
- **Registry:** PostgreSQL (metadata) + S3 (model artifacts)
- **Serving:** Docker containers, gunicorn + uvicorn workers
- **A/B Testing:** Traffic splitting via weighted routing
- **Monitoring:** Prometheus metrics, Evidently for drift
- **Queue:** Redis (for async batch predictions)
- **Deployment:** Docker Compose → ECS/EKS

### Key Features
1. **Model Registry** — Upload model (pickle/ONNX), tag version, store metadata (accuracy, training data hash)
2. **One-Click Deploy** — `POST /deploy {model_id, version, traffic_percent}`
3. **A/B Testing** — Route X% traffic to new model, compare metrics
4. **Shadow Mode** — New model gets requests but responses aren't served (compare offline)
5. **Auto-Rollback** — If new model's error rate > threshold, auto-revert
6. **Drift Detection** — Compare input feature distributions against training data

### Code Snippet: Model Serving Core
```python
from fastapi import FastAPI
import joblib
import numpy as np
from prometheus_client import Histogram, Counter

app = FastAPI()
PREDICTION_LATENCY = Histogram('prediction_latency_seconds', 'Time for prediction')
PREDICTION_COUNT = Counter('predictions_total', 'Total predictions', ['model_version'])

class ModelManager:
    def __init__(self):
        self.models = {}  # version -> model
        self.routing = {}  # version -> traffic_weight
    
    def load_model(self, version: str, path: str):
        self.models[version] = joblib.load(path)
    
    def predict(self, features: np.ndarray):
        version = self._select_version()
        model = self.models[version]
        with PREDICTION_LATENCY.time():
            prediction = model.predict(features)
        PREDICTION_COUNT.labels(model_version=version).inc()
        return {"prediction": prediction.tolist(), "model_version": version}
    
    def _select_version(self):
        # Weighted random selection for A/B testing
        import random
        versions = list(self.routing.keys())
        weights = list(self.routing.values())
        return random.choices(versions, weights=weights, k=1)[0]
```

---

## Project 3: Event-Driven Notification Engine

### What It Does
A scalable notification service that handles millions of notifications across email, SMS, push, and in-app channels with user preferences, rate limiting, batching, and delivery tracking.

### Why It's Impressive
Notification systems are asked in 30%+ of system design interviews. Building one proves you understand queues, priorities, idempotency, and multi-channel delivery.

### Architecture
```
Event Producers (Any Service)
         │
         ▼
┌─────────────────────┐
│   Event Bus (SQS)   │  ← Decoupled from producers
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Notification Router │  ← Check user preferences
│  - Channel selection │     Rate limiting per user
│  - Priority queue    │     Template rendering
│  - Deduplication     │     Batching (digest mode)
└────────┬────────────┘
         │
    ┌────┼────┬────────┐
    ▼    ▼    ▼        ▼
 Email  SMS  Push   In-App
 (SES) (SNS) (FCM) (WebSocket)
         │
         ▼
┌─────────────────────┐
│  Delivery Tracker    │  ← Track: sent, delivered, read, failed
│  - Retry on failure  │     Exponential backoff
│  - Dead letter queue │     Analytics dashboard
└─────────────────────┘
```

### Tech Stack
- **API:** FastAPI (send notification, manage preferences)
- **Queue:** AWS SQS (main) + Redis (rate limiting)
- **Channels:** AWS SES (email), AWS SNS (SMS), Firebase (push)
- **Storage:** PostgreSQL (preferences, templates, logs), Redis (dedup, rate limit)
- **Real-time:** WebSocket (in-app notifications)
- **Monitoring:** CloudWatch metrics, custom dashboard

### Key Features
1. **Multi-Channel Delivery** — Same event can trigger email + push based on user preference
2. **Template Engine** — Jinja2 templates with personalization variables
3. **Rate Limiting** — Max N notifications per user per hour (prevent spam)
4. **Digest Mode** — Batch low-priority notifications into daily digest
5. **Priority Queue** — Critical (password reset) > High (payment) > Low (marketing)
6. **Delivery Tracking** — Full lifecycle: queued → sent → delivered → opened → clicked
7. **Retry with Backoff** — Exponential retry for transient failures, dead letter for permanent

---

## Project 4: Distributed Job Scheduler (Mini Airflow)

### What It Does
A lightweight job scheduler that executes Python tasks as DAGs (directed acyclic graphs) with dependency management, retry logic, scheduling, and a web UI.

### Why It's Impressive
Shows deep understanding of distributed systems concepts: coordination, failure handling, idempotency, and state machines. This is infrastructure that companies like Airbnb and Uber build internally.

### Architecture
```
┌──────────────────┐     ┌──────────────────┐
│  Web UI (React)  │────►│  API Server      │
│  - DAG viewer    │     │  (FastAPI)       │
│  - Run history   │     │  - CRUD DAGs     │
│  - Logs          │     │  - Trigger runs  │
└──────────────────┘     └────────┬─────────┘
                                  │
                         ┌────────▼─────────┐
                         │   Scheduler       │
                         │   - Cron triggers │
                         │   - Dependency    │
                         │     resolution    │
                         └────────┬─────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼              ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Worker 1 │ │ Worker 2 │ │ Worker 3 │
              │ (execute │ │          │ │          │
              │  tasks)  │ │          │ │          │
              └──────────┘ └──────────┘ └──────────┘
                    │
                    ▼
           ┌──────────────┐
           │ PostgreSQL    │  ← Task state, run history
           │ Redis         │  ← Task queue, distributed lock
           └──────────────┘
```

### Tech Stack
- **Scheduler:** Python, APScheduler / custom cron parser
- **API:** FastAPI
- **Workers:** Celery or custom worker with Redis queue
- **Storage:** PostgreSQL (DAG definitions, run history), Redis (task queue, locks)
- **Frontend:** React (DAG visualization using D3.js or ReactFlow)
- **Deployment:** Docker Compose

### Key Features
1. **DAG Definition** — Define tasks and dependencies in Python/YAML
2. **Cron Scheduling** — Trigger DAGs on schedule
3. **Dependency Resolution** — Task B only runs when Task A succeeds
4. **Retry Logic** — Configurable retries with exponential backoff
5. **Distributed Locking** — Prevent duplicate task execution (Redis locks)
6. **Worker Heartbeat** — Detect dead workers, reassign their tasks
7. **Web UI** — Visualize DAG, see run history, view logs, manual trigger

---

## Project 5: Real-Time Fraud Detection Pipeline

### What It Does
A streaming system that scores financial transactions in real-time using ML features (velocity, geolocation, device fingerprint) and blocks suspicious ones within 100ms.

### Why It's Impressive
Combines streaming data engineering + ML + low-latency requirements. Directly relevant to fintech roles (which pay top salaries).

### Tech Stack
- **Streaming:** Kafka + PySpark Structured Streaming (or Flink)
- **Feature Store:** Redis (real-time features), PostgreSQL (historical)
- **ML Model:** XGBoost/LightGBM served via FastAPI
- **Rules Engine:** Python-based configurable rules (velocity, amount threshold)
- **Storage:** S3 (raw events), Redshift (analytics)
- **Deployment:** Docker, AWS ECS

### Key Features
1. **Real-Time Feature Computation** — User's transaction count in last 5 min / 1 hour / 24 hours
2. **ML Scoring** — Trained model scores each transaction (0-1 risk score)
3. **Rules Engine** — Configurable business rules (amount > $10K from new device → review)
4. **Decision Engine** — Combine ML score + rules → approve / review / block
5. **Feedback Loop** — Analyst marks false positives → model retraining data
6. **Latency SLA** — P99 < 100ms for decision

---

## Project 6: API Analytics & Observability Platform

### What It Does
Drop-in middleware that captures API request/response metadata, provides real-time dashboards, detects anomalies in traffic patterns, and generates insights.

### Why It's Impressive
Observability is critical at every company. Building a mini-Datadog shows you understand production systems deeply.

### Tech Stack
- **Collection:** FastAPI middleware (captures metrics)
- **Ingestion:** Kafka/Kinesis (high-throughput event stream)
- **Processing:** PySpark (aggregation, anomaly detection)
- **Storage:** TimescaleDB (time-series metrics), S3 (raw logs)
- **Dashboard:** Grafana
- **Alerting:** PagerDuty / Slack integration

### Key Features
1. **Zero-Config Middleware** — `app.add_middleware(APIAnalytics)` and it works
2. **Metrics:** Latency (P50/P95/P99), error rate, throughput, payload size
3. **Anomaly Detection** — Alert on unusual error rate spikes or latency degradation
4. **Endpoint Profiling** — Which endpoints are slow? Which have errors?
5. **User Analytics** — Request patterns per API key / user
6. **Cost Estimation** — Estimate infra cost per endpoint based on usage

---

## Project 7: Intelligent Document Processing Pipeline

### What It Does
Processes unstructured documents (invoices, contracts, reports) using OCR + LLM extraction to pull structured data and load into a data warehouse.

### Why It's Impressive
Combines document AI + data engineering + LLMs. Extremely high demand in enterprise (insurance, healthcare, finance all need this).

### Tech Stack
- **OCR:** AWS Textract or Tesseract
- **LLM Extraction:** Claude API / GPT-4 for structured extraction
- **Pipeline:** AWS Step Functions or Airflow
- **Storage:** S3 (documents), PostgreSQL (extracted data), pgvector (embeddings)
- **API:** FastAPI (upload, query, status)
- **Frontend:** React (upload interface, results viewer)

### Key Features
1. **Multi-Format Support** — PDF, images, scanned documents
2. **LLM-Powered Extraction** — Extract fields based on configurable schema
3. **Confidence Scoring** — Flag low-confidence extractions for human review
4. **Human-in-the-Loop** — Review queue for uncertain extractions
5. **Template Learning** — After processing 10+ similar docs, auto-detect layout
6. **Search** — Semantic search across all processed documents (embeddings + pgvector)

---

## Project 8: Multi-Tenant Feature Flag Service

### What It Does
A feature flag system that lets engineering teams control feature rollouts, A/B tests, and kill switches in real-time without deployments.

### Why It's Impressive
Shows you understand deployment best practices used at every FAANG company. Simple concept but the devil is in the details (caching, evaluation performance, targeting rules).

### Tech Stack
- **API:** FastAPI (flag CRUD, evaluation endpoint)
- **SDK:** Python SDK for server-side evaluation
- **Storage:** PostgreSQL (flag definitions, targeting rules)
- **Cache:** Redis (flag evaluation cache, <5ms response)
- **Frontend:** React (flag management dashboard)
- **Real-time:** WebSocket (instant flag updates to connected SDKs)

### Key Features
1. **Targeting Rules** — Enable for specific users, percentages, environments
2. **Gradual Rollout** — 1% → 5% → 25% → 100% with monitoring
3. **Kill Switch** — Instant disable without deployment
4. **Audit Log** — Who changed what, when
5. **SDK with Local Cache** — Evaluate flags locally, sync periodically

---

## Project 9: Data Lakehouse with Delta Lake

### What It Does
A mini data lakehouse on AWS that supports both batch analytics AND real-time queries on the same data using Delta Lake format.

### Why It's Impressive
Data lakehouse is THE architecture trend (Databricks, Snowflake). Shows you understand modern data architecture beyond basic ETL.

### Tech Stack
- **Storage:** S3 + Delta Lake format
- **Processing:** PySpark (batch ETL + streaming)
- **Catalog:** AWS Glue Data Catalog (or custom Hive Metastore)
- **Query:** Redshift Spectrum / Athena for SQL analytics
- **Orchestration:** Airflow / Step Functions
- **Quality:** Great Expectations (data quality checks)

### Key Features
1. **Bronze/Silver/Gold Layers** — Raw → cleaned → business-ready
2. **ACID Transactions** — On S3 data (Delta Lake)
3. **Time Travel** — Query data as it was at any point in time
4. **Schema Evolution** — Handle upstream schema changes gracefully
5. **Streaming + Batch Unified** — Same table supports both
6. **Data Quality Gates** — Promotion from bronze → silver requires quality checks

---

## Project 10: Conversational AI Agent with Tool Use

### What It Does
An AI assistant that can take actions — query databases, call APIs, execute code, search documents — orchestrated through a planning loop.

### Why It's Impressive
AI Agents are the frontier (2025-2026). Building one shows you're ahead of the curve. Directly relevant to AI-first companies.

### Tech Stack
- **LLM:** Claude API (via Anthropic SDK)
- **Framework:** Custom agent loop (or LangGraph)
- **Tools:** SQL executor, API caller, code interpreter, web searcher
- **Memory:** pgvector for long-term context
- **API:** FastAPI (chat endpoint)
- **Frontend:** React chat interface with tool execution display

### Key Features
1. **Planning Loop** — Agent breaks complex tasks into steps
2. **Tool Registry** — Pluggable tools with JSON schema definitions
3. **Memory** — Remembers conversation context + user preferences
4. **Guardrails** — Rate limiting, content filtering, cost caps
5. **Observability** — Log every LLM call, tool invocation, token usage
6. **Evaluation** — Test suite measuring accuracy on known queries

---

## How to Choose Your First Project

| Your Goal | Best Project | Why |
|-----------|-------------|-----|
| Data Engineer at FAANG | #1 (Data Quality) or #9 (Lakehouse) | Core DE skills + modern architecture |
| ML Engineer | #2 (Model Serving) or #5 (Fraud Detection) | MLOps + production ML |
| Backend/Platform | #4 (Job Scheduler) or #8 (Feature Flags) | Distributed systems depth |
| AI/GenAI roles | #7 (Document Processing) or #10 (AI Agent) | LLM applications |
| Generalist SDE at FAANG | #3 (Notifications) or #6 (Observability) | System design in code |

---

## Professional Quality Checklist

Before putting ANY project on your resume:

- [ ] Has a clear README with architecture diagram
- [ ] Docker Compose for one-command local setup
- [ ] At least integration tests for critical paths
- [ ] Error handling (not just happy path)
- [ ] Logging (structured, with request IDs)
- [ ] Metrics exposed (Prometheus format)
- [ ] Configuration via environment variables (not hardcoded)
- [ ] API documentation (Swagger/OpenAPI)
- [ ] Git history shows incremental progress (not one giant commit)
- [ ] Deployed somewhere accessible (or has clear deploy instructions)

---

## The "Ship in 3 Weeks" Rule

Don't spend 3 months on a project. Ship an MVP in 3 weeks:
- **Week 1:** Core functionality working end-to-end
- **Week 2:** Add the "wow" feature (ML, real-time, scale)
- **Week 3:** Polish (docs, tests, deployment, monitoring)

A shipped imperfect project beats a perfect unfinished one every time.
