# AI, ML & Modern Tech — What to Know and Why

## Why This Section Matters

In 2025-2026, EVERY tech company is investing in AI/ML. Even if you're not applying for a pure ML role, understanding these concepts:
- Makes you a stronger system design candidate
- Opens doors to ML Engineering / MLOps roles
- Shows technical curiosity (a hiring signal)

---

## ML Fundamentals (What Every Engineer Should Know)

### The ML Landscape
```
Machine Learning
├── Supervised Learning (labeled data)
│   ├── Classification (spam/not spam, cat/dog)
│   └── Regression (predict price, temperature)
├── Unsupervised Learning (no labels)
│   ├── Clustering (customer segments)
│   └── Dimensionality Reduction (PCA)
├── Reinforcement Learning (reward-based)
│   └── Agent learns through trial & error (games, robotics)
└── Deep Learning (neural networks)
    ├── CNNs (images)
    ├── RNNs/LSTMs (sequences)
    └── Transformers (text, everything now)
```

### Key Concepts
| Concept | One-Line Explanation | Why It Matters |
|---------|---------------------|----------------|
| Overfitting | Model memorizes training data, fails on new data | Always split: train/val/test |
| Underfitting | Model too simple to capture patterns | Need more features or complexity |
| Bias-Variance | Simple models = high bias, complex = high variance | Find the sweet spot |
| Feature Engineering | Transform raw data into useful inputs | Often more impactful than model choice |
| Cross-validation | K-fold split for reliable evaluation | Especially with small datasets |
| Gradient Descent | Iteratively minimize error by following gradient | Core of all neural network training |
| Regularization | L1/L2 penalty to prevent overfitting | Standard practice |

### Evaluation Metrics
```
Classification:
- Accuracy: Overall correct predictions (misleading for imbalanced data)
- Precision: Of predicted positives, how many are correct? (minimize false positives)
- Recall: Of actual positives, how many did we find? (minimize false negatives)
- F1: Harmonic mean of precision and recall
- AUC-ROC: Model's ability to distinguish classes at all thresholds

Regression:
- MSE / RMSE: Average squared error
- MAE: Average absolute error
- R²: Proportion of variance explained

When to use what:
- Fraud detection → Precision (don't annoy legitimate users)
- Cancer screening → Recall (don't miss actual cases)
- General → F1 (balanced)
```

---

## Generative AI & LLMs

### What LLMs Are
Large Language Models (GPT-4, Claude, Gemini) are transformer-based models trained on massive text data. They predict the next token, but this simple objective produces remarkable reasoning ability.

### Key Concepts
| Term | Meaning | Why It Matters |
|------|---------|----------------|
| Transformer | Architecture based on self-attention | Foundation of all modern AI |
| Tokenization | Text → numbers (subword units) | Determines input cost and limitations |
| Context Window | Max tokens the model can process | Affects what information model can use |
| Temperature | Randomness in generation (0=deterministic, 1=creative) | Controls output diversity |
| Prompt Engineering | Crafting input to get better output | Practical skill for building AI apps |
| Fine-tuning | Training on domain-specific data | Adapt model to your use case |
| RAG (Retrieval-Augmented Generation) | Feed relevant documents to LLM | Ground responses in your data |
| Embeddings | Dense vector representation of text | Enable semantic search, similarity |
| Hallucination | Model generates plausible but false information | Critical limitation to handle |
| Agents | LLMs that can take actions (call tools, APIs) | The frontier of AI applications |

### RAG Architecture (Most Common AI Pattern in Industry)
```
User Query
    │
    ▼
┌──────────────┐     ┌──────────────────┐
│ Embed Query  │────►│ Vector Database   │
└──────────────┘     │ (search similar   │
                     │  documents)       │
                     └────────┬─────────┘
                              │ Top K relevant chunks
                              ▼
                     ┌──────────────────┐
                     │ Construct Prompt  │
                     │ Query + Context   │
                     └────────┬─────────┘
                              │
                              ▼
                     ┌──────────────────┐
                     │ LLM (Claude/GPT) │
                     │ Generate Answer   │
                     └────────┬─────────┘
                              │
                              ▼
                     Grounded Response

Tools: LangChain, LlamaIndex, Pinecone, Weaviate, pgvector
```

### Building AI Applications (What Employers Want)
```
Not just: "I used the OpenAI API"
But: "I built a production RAG system with:
  - Chunking strategy (semantic vs fixed-size)
  - Embedding model selection (trade-off: cost vs quality)
  - Vector DB with metadata filtering
  - Evaluation pipeline (faithfulness, relevance)
  - Guardrails (content filtering, cost limits)
  - Observability (token usage, latency, quality metrics)"
```

---

## Cloud Computing (AWS Focus)

### Services You Must Know

| Category | Service | Purpose | Interview Relevance |
|----------|---------|---------|-------------------|
| Compute | EC2 | Virtual machines | Basic infrastructure |
| Compute | Lambda | Serverless functions | Event-driven architectures |
| Compute | ECS/EKS | Container orchestration | Microservices deployment |
| Storage | S3 | Object storage (files, images) | Almost every system design |
| Storage | EBS | Block storage for EC2 | Database storage |
| Database | RDS | Managed relational DB | CRUD applications |
| Database | DynamoDB | Managed NoSQL | High-scale key-value |
| Database | ElastiCache | Managed Redis/Memcached | Caching layer |
| Messaging | SQS | Message queue | Async processing |
| Messaging | SNS | Pub/Sub notifications | Fan-out events |
| Messaging | Kinesis | Real-time data streaming | Log/event processing |
| Networking | CloudFront | CDN | Static content delivery |
| Networking | Route 53 | DNS + routing | Global load balancing |
| Networking | ALB/NLB | Load balancing | Traffic distribution |
| Data | Redshift | Data warehouse | Analytics/BI |
| Data | Glue | ETL service | Data pipelines |
| ML | SageMaker | ML model training & deployment | ML workloads |

### AWS Architecture Patterns (System Design)
```
Pattern 1: Three-Tier Web App
  Route 53 → CloudFront → ALB → EC2/ECS → RDS + ElastiCache

Pattern 2: Event-Driven Microservices
  API Gateway → Lambda → SQS → Lambda → DynamoDB
                              → SNS → Multiple subscribers

Pattern 3: Data Pipeline
  Sources → Kinesis → Lambda → S3 → Glue → Redshift → QuickSight

Pattern 4: ML Pipeline
  S3 (data) → SageMaker (train) → Lambda (inference endpoint) → API Gateway
```

---

## MLOps — The Hot Skill

### What MLOps Is
DevOps for machine learning. Taking models from notebook to production reliably.

### MLOps Lifecycle
```
1. Data Management → Version data (DVC), track lineage
2. Experiment Tracking → Log metrics, params, artifacts (MLflow, W&B)
3. Training Pipeline → Reproducible, automated (Kubeflow, SageMaker Pipelines)
4. Model Registry → Store versioned models with metadata
5. Deployment → Serve predictions (batch or real-time)
6. Monitoring → Track data drift, model performance decay
7. Retraining → Trigger when performance drops
```

### Key Tools
| Purpose | Tools |
|---------|-------|
| Experiment Tracking | MLflow, Weights & Biases, Neptune |
| Pipeline Orchestration | Airflow, Kubeflow, Prefect |
| Model Serving | TensorFlow Serving, TorchServe, BentoML, FastAPI |
| Feature Store | Feast, Tecton |
| Monitoring | Evidently, WhyLabs, Arize |
| Data Versioning | DVC, LakeFS |

---

## What to Learn and Why (Priority for Your Career)

### Tier 1: Learn Now (Directly Hireable)
| Skill | Why | How to Learn |
|-------|-----|-------------|
| RAG + LLM Apps | Every company is building these | Build a project with LangChain + vector DB |
| Cloud (AWS/GCP) | Infrastructure for everything | Get AWS SAA cert (you have it!), build projects |
| Python for ML | The language of data/ML | scikit-learn, pandas, numpy mastery |
| SQL + Data Engineering | Foundation of all ML | PySpark, Glue, Redshift (your strength) |

### Tier 2: Learn Soon (Career Multiplier)
| Skill | Why | How to Learn |
|-------|-----|-------------|
| MLOps (MLflow, Docker, K8s) | Gap between ML and production | Deploy a model end-to-end |
| Prompt Engineering | Practical AI skill | Build complex AI workflows |
| Vector Databases | Core of modern search/RAG | Use Pinecone or pgvector in a project |
| Fine-tuning LLMs | Customize for domains | LoRA/QLoRA on open-source models |

### Tier 3: Stay Aware (Future Advantage)
| Skill | Why | Timeline |
|-------|-----|----------|
| AI Agents | Next wave of applications | 2025-2026 |
| Multimodal AI | Vision + Text + Audio | Growing rapidly |
| Edge AI | On-device inference | IoT, mobile |
| Synthetic Data | Training without real data | Privacy-preserving ML |

---

## Interview Questions on AI/ML

**Q: How would you deploy an ML model to production?**
"I'd containerize with Docker, serve via FastAPI or SageMaker endpoint, put behind a load balancer, add monitoring for latency and data drift, version the model in a registry, and set up A/B testing before full rollout."

**Q: What's the difference between batch and real-time inference?**
- Batch: Process accumulated data periodically (cheaper, higher throughput). Use for: recommendations, reports.
- Real-time: Process each request immediately (more infra, lower latency). Use for: fraud detection, chatbots.

**Q: How do you handle model degradation?**
"Monitor prediction distributions and compare against training data. Set alerts for drift. Retrain on fresh data when performance drops below threshold. Shadow mode for new models before switching traffic."

**Q: Explain RAG to a non-technical stakeholder.**
"Instead of asking the AI to answer from memory (which can be wrong), we first find relevant documents from our knowledge base, then ask the AI to answer based specifically on those documents. Like giving a human expert the right reference materials before asking a question."
