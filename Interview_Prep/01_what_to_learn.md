# What to Learn — Implementation & Design Round

## Priority Matrix (Based on Your Resume)

### P0 — Must Know (Will definitely be asked)

| Topic | Why | What to Cover |
|-------|-----|---------------|
| **System Design (HLD)** | Core of design round | Load balancers, DB sharding, caching, message queues, microservices, CAP theorem |
| **Low-Level Design (LLD)** | Implementation round tests this | SOLID principles, design patterns (Factory, Strategy, Observer, Singleton), class diagrams |
| **Python Deep Dive** | Your primary language | Decorators, generators, metaclasses, GIL, async/await, memory management, concurrency |
| **SQL Advanced** | You work with 125M+ records | Window functions, CTEs, query optimization, indexing strategies, explain plans |
| **API Design** | FastAPI/Django on resume | REST best practices, versioning, pagination, rate limiting, authentication, error handling |
| **ETL Pipeline Design** | Your daily work at TCS | Idempotency, exactly-once processing, backpressure, dead letter queues, schema evolution |

### P1 — Should Know (High probability)

| Topic | Why | What to Cover |
|-------|-----|---------------|
| **AWS Architecture** | Certified + daily use | Well-Architected Framework, serverless patterns, event-driven architecture, cost optimization |
| **Data Modeling** | Working with HCP records | Star schema, snowflake, normalization vs denormalization, slowly changing dimensions |
| **WebSocket/Real-time** | Chat app project | Connection lifecycle, scaling WebSockets, pub/sub patterns, fallback strategies |
| **ML System Design** | AI/ML Engineer role | Feature stores, model serving, A/B testing, monitoring, retraining pipelines |
| **Distributed Systems** | Scaling ETL pipelines | Consistency models, partitioning, replication, consensus (Raft/Paxos basics) |

### P2 — Good to Know (Differentiators)

| Topic | Why | What to Cover |
|-------|-----|---------------|
| **Docker & Kubernetes** | MLOps goal | Containerization, orchestration, service mesh, health checks |
| **CI/CD for ML** | MLOps pipeline | MLflow, model registry, automated testing for ML, canary deployments |
| **Stream Processing** | Complements batch ETL | Kafka, Spark Streaming, event sourcing, CQRS |
| **GenAI/LLM Architecture** | Trending + portfolio goal | RAG, vector databases, prompt engineering, agent frameworks |

---

## Study Plan — Topic Breakdown

### 1. System Design (HLD) — 5 Days

**Day 1:** Fundamentals
- Scalability (vertical vs horizontal)
- Load balancing (Round Robin, Least Connections, Consistent Hashing)
- CDN, DNS, Reverse Proxy

**Day 2:** Data Layer
- SQL vs NoSQL trade-offs
- Database replication (master-slave, multi-master)
- Sharding strategies (hash-based, range-based, geo-based)
- Connection pooling

**Day 3:** Caching & Messaging
- Cache strategies (write-through, write-back, write-around)
- Cache invalidation patterns
- Message queues (SQS, Kafka, RabbitMQ)
- Pub/Sub vs Point-to-Point

**Day 4:** Microservices & APIs
- Service decomposition
- API Gateway pattern
- Service discovery
- Circuit breaker, retry, timeout patterns
- gRPC vs REST vs GraphQL

**Day 5:** Practice Designs
- Design a data pipeline (ETL system)
- Design a real-time chat system
- Design a URL shortener
- Design a notification system

---

### 2. Low-Level Design (LLD) — 4 Days

**Day 1:** OOP & SOLID
- Single Responsibility, Open/Closed, Liskov, Interface Segregation, Dependency Inversion
- Practice: Identify violations in code snippets

**Day 2:** Design Patterns
- Creational: Factory, Builder, Singleton
- Structural: Adapter, Decorator, Facade
- Behavioral: Strategy, Observer, Command, State

**Day 3:** LLD Problems
- Design a Parking Lot
- Design an Elevator System
- Design a File Storage System (like Google Drive)
- Design a Rate Limiter class

**Day 4:** Code Architecture
- Clean Architecture layers
- Repository pattern
- Dependency Injection
- Event-driven architecture in code

---

### 3. Python Implementation — 3 Days

**Day 1:** Advanced Python
```python
# Decorators with arguments
def retry(max_attempts=3, delay=1):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(delay * (2 ** attempt))
        return wrapper
    return decorator

# Context managers
class DatabaseConnection:
    def __enter__(self):
        self.conn = create_connection()
        return self.conn
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.conn.close()
        return False

# Generators for large data
def process_large_file(filepath):
    with open(filepath) as f:
        for line in f:
            yield transform(line)
```

**Day 2:** Concurrency & Async
- Threading vs Multiprocessing vs Asyncio
- GIL implications
- `async/await` with FastAPI
- Connection pools, semaphores

**Day 3:** Data Structures Implementation
- Implement LRU Cache
- Implement a Thread-safe Queue
- Implement a Trie
- Implement a Bloom Filter

---

### 4. SQL & Data Engineering — 2 Days

**Day 1:** Advanced SQL
```sql
-- Window functions
SELECT 
    patient_id,
    visit_date,
    diagnosis,
    ROW_NUMBER() OVER (PARTITION BY patient_id ORDER BY visit_date DESC) as visit_rank,
    LAG(visit_date) OVER (PARTITION BY patient_id ORDER BY visit_date) as prev_visit
FROM hcp_records;

-- CTEs for complex queries
WITH daily_metrics AS (
    SELECT date, COUNT(*) as records_processed
    FROM etl_log
    GROUP BY date
),
anomalies AS (
    SELECT date, records_processed,
           AVG(records_processed) OVER (ORDER BY date ROWS 7 PRECEDING) as rolling_avg
    FROM daily_metrics
)
SELECT * FROM anomalies WHERE records_processed < rolling_avg * 0.5;
```

**Day 2:** ETL & Data Pipeline Concepts
- Batch vs Stream processing
- Data quality checks (Great Expectations pattern)
- Schema evolution strategies
- Slowly Changing Dimensions (Type 1, 2, 3)
- Partitioning strategies in S3/Redshift

---

### 5. AWS Architecture — 2 Days

**Day 1:** Core Services Deep-dive
- Lambda: cold starts, concurrency, layers
- Glue: job bookmarks, pushdown predicates, DPU tuning
- Redshift: distribution keys, sort keys, VACUUM, workload management
- S3: lifecycle policies, event notifications, cross-region replication

**Day 2:** Architecture Patterns
- Event-driven: S3 → Lambda → SQS → Lambda → Redshift
- Serverless ETL: Glue + Step Functions + CloudWatch
- Cost optimization strategies you've actually used
- Well-Architected Framework pillars

---

### 6. ML System Design — 2 Days (if AI/ML role)

**Day 1:** ML Pipeline Architecture
- Feature store design
- Training pipeline (data → features → model → evaluation)
- Model serving (batch vs real-time)
- A/B testing framework

**Day 2:** MLOps
- Model versioning (MLflow)
- Monitoring: data drift, concept drift, model degradation
- CI/CD for ML: test data quality, model performance gates
- Infrastructure: SageMaker vs self-hosted

---

## Key Resources to Revisit in This Repo

| Topic | Files |
|-------|-------|
| System Design Fundamentals | `opus/system_design/fundamentals/` |
| Design Problems | `opus/system_design/designs/` |
| DSA (if needed) | `opus/dsa/` |
| Interview Strategy | `opus/career_guide/05_interview_strategy/` |
| Cheatsheets | `opus/career_guide/06_cheatsheets/` |

---

## Gap Analysis (What's Missing from Your Resume)

| Gap | Impact | Quick Fix |
|-----|--------|-----------|
| No LLD/Design Patterns mentioned | They WILL ask | Study SOLID + 5 key patterns |
| No Docker/K8s | Expected for senior roles | Learn basic containerization |
| No testing mentioned | Shows maturity | Add pytest, unit test examples |
| No CI/CD experience shown | Expected at senior level | Learn GitHub Actions basics |
| No streaming experience | Complements batch ETL | Understand Kafka basics |

Focus your energy: **60% on Design (HLD+LLD), 25% on Implementation (Python+SQL), 15% on AWS/ML specifics.**
