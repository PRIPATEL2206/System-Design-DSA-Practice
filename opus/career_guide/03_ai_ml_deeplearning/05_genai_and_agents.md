# GenAI, LLMs & AI Agents — The 2025 Revolution

## Why This Is THE Skill to Have
Every tech company is building with LLMs. This is the fastest-growing demand area in all of engineering. Understanding GenAI puts you in the top 1% of candidates.

---

## 1. How LLMs Work (Transformer Architecture)

```
Input: "The cat sat on the"  →  Output: "mat" (next token prediction)

Transformer Architecture (simplified):
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Input tokens → Embedding → Positional Encoding                 │
│                                    │                             │
│                    ┌───────────────┼───────────────┐            │
│                    │    Attention Block (×N layers)  │            │
│                    │                               │            │
│                    │  ┌─────────────────────────┐  │            │
│                    │  │  Multi-Head Attention    │  │            │
│                    │  │  Q × K^T / √d_k → softmax│ │            │
│                    │  │  → × V                   │  │            │
│                    │  └────────────┬────────────┘  │            │
│                    │               │               │            │
│                    │  ┌────────────▼────────────┐  │            │
│                    │  │  Feed-Forward Network    │  │            │
│                    │  │  (MLP: expand + contract)│  │            │
│                    │  └────────────┬────────────┘  │            │
│                    │               │               │            │
│                    └───────────────┼───────────────┘            │
│                                    │ (repeat N times)           │
│                                    ▼                             │
│                    Linear → Softmax → Next Token Probability    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Key Innovation: Self-Attention
  "The cat sat on the mat because it was tired"
  Attention lets "it" attend to "cat" (understands reference!)
  
  Attention(Q, K, V) = softmax(QK^T / √d_k) × V
  
  Each token asks: "How relevant is every other token to me?"
```

### Model Scale
```
Model            Parameters    Training Data    Training Cost
──────────────────────────────────────────────────────────────
GPT-2            1.5B         40GB text         ~$50K
GPT-3            175B         570GB text        ~$5M
GPT-4            ~1.8T (MoE)  13T tokens       ~$100M
Claude 3.5       Unknown      Massive           Unknown
Llama 3 (70B)    70B          15T tokens       ~$10M
Gemini Ultra     Unknown      Multimodal        Unknown

MoE (Mixture of Experts): Only ~10% of parameters active per token
  → Larger model capacity with same compute cost
```

---

## 2. Prompt Engineering

```
The skill of getting the best output from LLMs through input design.

Levels of Prompting:
────────────────────
Zero-shot:  "Translate this to French: Hello"
Few-shot:   "English: Hello → French: Bonjour\nEnglish: Goodbye → French: ?"
Chain-of-Thought: "Let's think step by step..."
ReAct:      Reasoning + Acting (think, then call a tool)

Advanced Techniques:
────────────────────
System prompt:       Set persona, rules, format
Temperature:         0 = deterministic, 1 = creative
Top-p (nucleus):     Sample from top probability mass
Few-shot examples:   Show desired input/output format
Structured output:   "Return JSON with fields: name, age, city"
Self-consistency:    Ask multiple times, take majority answer
Tree-of-Thought:     Explore multiple reasoning paths
```

### Prompt Template (Production)
```
SYSTEM: You are a senior Python developer. Return only code, no explanations.
         Follow PEP 8. Use type hints. Handle edge cases.

USER: Write a function that {task_description}

Input format: {input_spec}
Output format: {output_spec}
Constraints: {constraints}

Examples:
Input: {example_input}
Output: {example_output}
```

---

## 3. RAG (Retrieval-Augmented Generation)

```
Problem: LLMs hallucinate and don't know your private data.
Solution: Retrieve relevant documents, inject into prompt.

┌──────────────────── RAG Pipeline ────────────────────────────┐
│                                                               │
│  Indexing (offline):                                         │
│  Documents → Chunk (500 tokens) → Embed → Vector DB          │
│                                                               │
│  Query (online):                                             │
│  User question → Embed → Vector search → Top K chunks        │
│       │                                        │             │
│       │         ┌─────────────────────────────┘             │
│       │         │                                            │
│       ▼         ▼                                            │
│  LLM Prompt = "Given this context: {chunks}\n                │
│                Answer this question: {user_question}"        │
│       │                                                      │
│       ▼                                                      │
│  Grounded answer (with citations!)                           │
└──────────────────────────────────────────────────────────────┘

Key Components:
  Embedding model: text-embedding-3-large, cohere-embed
  Vector DB: Pinecone, Weaviate, Qdrant, Chroma, pgvector
  Chunking: Fixed size, semantic splitting, recursive
  Retrieval: Cosine similarity, hybrid (dense + sparse)
  Reranking: Cross-encoder to rerank top candidates

Advanced RAG:
  - HyDE: Generate hypothetical answer, use IT for retrieval
  - Multi-query: Rephrase question multiple ways for better recall
  - Parent document retrieval: Return larger context than chunk
  - Self-RAG: LLM decides when to retrieve
  - GraphRAG: Knowledge graph + vector retrieval
```

---

## 4. Fine-Tuning LLMs

```
When to fine-tune vs prompt:
─────────────────────────────
Prompt engineering: Quick, cheap, good for general tasks
Fine-tuning: When you need consistent style, domain expertise, or cost reduction

Types of Fine-tuning:
─────────────────────
Full fine-tuning:   Update ALL parameters (expensive, needs lots of data)
LoRA:              Low-Rank Adaptation — add small trainable matrices
                    Only ~0.1% parameters trained! 10x cheaper.
QLoRA:             LoRA + Quantization (4-bit) — train on single GPU!
RLHF:             Reinforcement Learning from Human Feedback
                    Used to align ChatGPT (preference learning)
DPO:               Direct Preference Optimization (simpler than RLHF)

Fine-tuning Stack:
  - Hugging Face Transformers + PEFT (LoRA)
  - Axolotl (easy fine-tuning framework)
  - Unsloth (2x faster LoRA training)
  - LLaMA Factory (GUI-based)
```

---

## 5. AI Agents

```
Agent = LLM + Tools + Memory + Planning

┌────────────────────────────────────────────────────────────┐
│                        AI Agent                             │
│                                                             │
│  ┌─────────┐   ┌──────────┐   ┌──────────┐   ┌────────┐ │
│  │   LLM   │──▶│ Planning │──▶│Tool Call │──▶│ Memory │ │
│  │ (brain) │   │ (decide  │   │(execute) │   │(learn) │ │
│  │         │◀──│  next)   │◀──│          │◀──│        │ │
│  └─────────┘   └──────────┘   └──────────┘   └────────┘ │
│                                     │                      │
│                         ┌───────────┼───────────┐         │
│                         │           │           │         │
│                    ┌────▼──┐   ┌───▼────┐  ┌──▼─────┐  │
│                    │Search │   │Database│  │Code    │  │
│                    │API    │   │Query   │  │Execute │  │
│                    └───────┘   └────────┘  └────────┘  │
└────────────────────────────────────────────────────────────┘

Agent Frameworks:
  - LangChain / LangGraph (most popular)
  - CrewAI (multi-agent collaboration)
  - AutoGen (Microsoft, multi-agent)
  - Claude Tool Use (Anthropic native)
  - OpenAI Assistants API

Agent Patterns:
  ReAct: Think → Act → Observe → Think → ...
  Plan-and-Execute: Make full plan → execute steps
  Multi-Agent: Specialized agents collaborate
  Reflection: Agent critiques and improves its own output
```

---

## 6. Vector Databases & Embeddings

```
Embedding: Convert text/image → dense vector (768-3072 dimensions)
  "King" → [0.2, -0.5, 0.8, ...]  (captures meaning!)
  
  Similarity: cosine(embed("King"), embed("Queen")) ≈ 0.85
  
Vector DB stores millions of embeddings for fast similarity search.

Vector DB Comparison:
┌────────────────┬────────────────────────────────────────────────┐
│ Pinecone       │ Fully managed, easy, expensive                 │
│ Weaviate       │ Open source, hybrid search (vector + keyword)  │
│ Qdrant         │ Open source, fast, Rust-based                  │
│ Chroma         │ Lightweight, good for prototyping              │
│ pgvector       │ PostgreSQL extension (use existing DB!)        │
│ Milvus         │ Enterprise-grade, GPU-accelerated              │
└────────────────┴────────────────────────────────────────────────┘

Search algorithms:
  - Brute force: O(n) exact — too slow at scale
  - HNSW: Graph-based approximate NN — fast, memory-heavy
  - IVF: Inverted file index — cluster-based search
  - PQ: Product quantization — compressed vectors
```

---

## 7. LLM in Production

```
Challenges:
───────────
1. Latency: LLMs are slow (1-10s per response)
   → Streaming, caching, smaller models for simple tasks

2. Cost: GPT-4 = $30/1M tokens
   → Caching, prompt optimization, fine-tune smaller model

3. Hallucination: LLMs make up facts
   → RAG, citations, verification layer, constrained output

4. Safety: Prompt injection, harmful outputs
   → Input/output filters, guardrails, red-teaming

5. Evaluation: Hard to measure quality
   → LLM-as-judge, human eval, domain-specific metrics

Production Architecture:
┌────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
│ Client │──▶│ API Gateway│──▶│ Orchestrator│──▶│   LLM API  │
└────────┘   │ (rate limit)│   │ (routing,  │   │(Claude/GPT)│
             └────────────┘   │  caching)  │   └────────────┘
                              └──────┬─────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
              ┌─────▼─────┐  ┌──────▼──────┐  ┌────▼─────┐
              │ Vector DB  │  │  Prompt     │  │ Guardrails│
              │ (RAG)      │  │  Template   │  │ (safety)  │
              └────────────┘  └─────────────┘  └───────────┘
```

---

## 8. Key Skills to Build

```
For ML Engineer / AI Engineer roles at Big Tech:

Must-have:
  ✅ Python + PyTorch/TensorFlow
  ✅ Transformer architecture understanding
  ✅ RAG implementation (end-to-end)
  ✅ Prompt engineering (advanced)
  ✅ Fine-tuning (LoRA/QLoRA)
  ✅ Vector databases
  ✅ ML system design
  ✅ Evaluation metrics

Good-to-have:
  ✅ Agent frameworks (LangChain/LangGraph)
  ✅ MLOps (MLflow, model serving)
  ✅ Distributed training (DeepSpeed, FSDP)
  ✅ Multimodal models (vision + language)
  ✅ Reinforcement learning basics (RLHF)

Portfolio projects that stand out:
  1. RAG chatbot on custom knowledge base
  2. Fine-tuned model for specific domain
  3. Multi-agent system that solves complex task
  4. ML pipeline with monitoring and auto-retraining
```
