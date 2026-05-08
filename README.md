# System Design & DSA Practice

A personal repository tracking my journey from mid-level to senior ML/Data Engineer — through consistent system design practice and DSA problem solving.

---

## About

**Role:** System Engineer @ TCS | Targeting Senior ML/Data Engineer  
**Stack:** Python · PySpark · AWS · FastAPI · PostgreSQL · MongoDB  
**Goal:** Crack senior-level interviews at product companies (Flipkart, Swiggy, Razorpay, Amazon)

---

## Repository Structure

```
System-Design-DSA-Practice/
│
├── system-design/
│   ├── problems/
│   │   ├── fraud-detection-system.md
│   │   ├── log-ingestion-anomaly-detection.md
│   │   └── ...
│   ├── concepts/
│   │   ├── message-queues-kafka-rabbitmq-kinesis.md
│   │   ├── storage-tiering-hot-cold-archive.md
│   │   ├── lambda-limitations-at-scale.md
│   │   └── ...
│   └── templates/
│       └── system-design-template.md
│
├── dsa/
│   ├── arrays/
│   ├── dynamic-programming/
│   │   ├── max-path-score.py
│   │   └── ...
│   ├── graphs/
│   ├── trees/
│   └── sliding-window/
│
└── README.md
```

---

## System Design Problems

| # | Problem | Difficulty | Key Concepts | Status |
|---|---------|------------|--------------|--------|
| 1 | [Fraud Detection System](system-design/problems/fraud-detection-system.md) | Medium | Kafka, Lambda, SageMaker, MongoDB | In Progress |
| 2 | [Log Ingestion & Anomaly Detection](system-design/problems/log-ingestion-anomaly-detection.md) | Hard | Kinesis, Storage Tiering, Athena, Alert Dedup | In Progress |
| 3 | Real-Time ML Feature Store | Hard | Feature Store, Online/Offline, Dual Write | Pending |

---

## DSA Problems

| # | Problem | Platform | Topic | Difficulty | Status |
|---|---------|----------|-------|------------|--------|
| 1 | Max Path Score | LeetCode | Dynamic Programming | Medium | Solved |

---

## Concepts Studied

| Topic | Notes | Status |
|-------|-------|--------|
| Kafka vs RabbitMQ vs Kinesis | [Notes](system-design/concepts/message-queues-kafka-rabbitmq-kinesis.md) | Done |
| Storage Tiering (Hot/Warm/Cold) | Pending | Pending |
| Lambda Limits at Scale | Pending | Pending |
| Rate Limiting & Backpressure | Pending | Pending |
| Alert Design & Deduplication | Pending | Pending |

---

## System Design Template

Every problem is approached using this structure:

```
1. Requirements Clarification
   - Functional requirements
   - Non-functional (scale, latency, availability)

2. Capacity Estimation
   - Requests/sec, storage, bandwidth

3. High-Level Architecture
   - Component diagram

4. Deep Dive
   - Ingestion layer
   - Storage strategy (hot/cold/archive)
   - Processing layer
   - Query layer

5. Bottlenecks & Trade-offs
   - What breaks first under 10x load
   - How to fix it

6. Failure Scenarios
   - What if queue goes down?
   - What if DB is slow?
```

---

## Progress Tracker

| Week | System Design | DSA | Concept Studied |
|------|--------------|-----|-----------------|
| Week 1 (May 2026) | Fraud Detection, Log Ingestion | Max Path Score | Kafka vs RabbitMQ vs Kinesis |
| Week 2 | | | |
| Week 3 | | | |
| Week 4 | | | |

---

## Key Lessons Learned

### Storage
- Never store years of log data in MongoDB — use hot/cold tiering
- S3 + Athena = cheap SQL on billions of rows
- S3 Lifecycle Policy handles archival automatically — no Glue job needed

### Message Queues
- RabbitMQ = task queue (one consumer, message deleted after ACK)
- Kafka/Kinesis = event log (multiple consumers, replay anytime)
- Default to Kinesis on AWS, SQS for simple task queues

### Lambda at Scale
- Cold starts = 200-500ms — avoid for latency-sensitive paths
- Default concurrency = 1000 — hits limit at ~1000 req/sec
- Use Kinesis → Lambda batch trigger to absorb spikes

### Alerting
- Always deduplicate alerts — alert storms make engineers ignore notifications
- Route by severity: email → Slack → PagerDuty
- Use Redis TTL windows for deduplication (5-min window pattern)

---

## Resources

- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)
- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/)
- [LeetCode](https://leetcode.com/)

---

*Updated weekly as part of daily interview preparation.*
