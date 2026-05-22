# How to Answer Any System Design Question

## The 4-Step RADIO Framework

| Step | What to Do | Time |
|------|-----------|------|
| **R**equirements | Clarify what you're building and what you're NOT | 5 min |
| **A**rchitecture | Draw the high-level system diagram | 5 min |
| **D**ata | Define data model, storage choices | 5 min |
| **I**nterface | Define APIs, communication protocols | 5 min |
| **O**ptimize | Scale, handle bottlenecks, edge cases | 10 min |

---

## Step 1: Requirements (ALWAYS Do This First)

The interviewer will say "Design Twitter." Do NOT start drawing. Ask:

### Functional Requirements — what the system does
- "Should users be able to post tweets, or only read?"
- "Do we need to support images/video, or just text?"
- "Do we need search? DMs? Notifications?"
- "Is this read-heavy or write-heavy?"

### Non-Functional Requirements — how the system behaves
- "What scale are we targeting? DAU? QPS?"
- "What's the acceptable latency? (e.g., < 200ms for feed)"
- "What availability do we need? 99.9% or 99.99%?"
- "Does it need to be strongly consistent or is eventual OK?"
- "What geography? Global or one region?"

### Scope — draw a boundary
- "I'll focus on X and Y, and treat Z as out of scope. Does that work?"

---

## Step 2: Capacity Estimation

Do quick math. Shows you think like an engineer.

### Template
```
DAU = X million
Reads per user/day = Y
Writes per user/day = Z

Read QPS  = DAU × Y / 86400
Write QPS = DAU × Z / 86400
Peak QPS  = average × 2 to 5

Storage/day = Write QPS × avg_object_size × 86400
Storage/year = Storage/day × 365
```

### Key Numbers to Memorise (see 08_numbers_cheatsheet.md)
- 1 million req/day  ≈ 12 req/sec
- 1 billion req/day  ≈ 11,500 req/sec
- 1 KB × 1 million = 1 GB
- 1 KB × 1 billion = 1 TB

---

## Step 3: High-Level Architecture

Draw boxes and arrows. Every FAANG design uses some version of this:

```
[Client / Mobile]
       |
   [DNS]
       |
   [CDN] -----> (static assets served here)
       |
[Load Balancer]
       |
 +-----------+
 | App Server|  (stateless, horizontally scalable)
 +-----------+
       |
  +---------+         +-----------+
  |  Cache  | <-----> |  Database |
  | (Redis) |         | (Primary) |
  +---------+         +-----------+
                           |
                    [Read Replicas]
       |
[Message Queue]  (async tasks)
       |
[Worker / Consumer]
```

**Rule:** Always separate read path from write path once you've drawn the basic diagram.

---

## Step 4: Deep Dive — Component by Component

The interviewer will pick a component to drill into. Common areas:

| Component | What They Want to Hear |
|-----------|----------------------|
| Database | Schema, indexing, sharding key choice, SQL vs NoSQL reasoning |
| Cache | What to cache, eviction policy, how to handle cache miss |
| Load Balancer | Algorithm (round robin / consistent hash), sticky sessions |
| Message Queue | Why async here, ordering guarantees, retry logic |
| API | REST endpoints, pagination, rate limiting |

---

## Step 5: Scaling & Trade-offs

After the basic design is done, go through this checklist:

```
[ ] Where is the bottleneck? (usually DB reads)
[ ] How to scale the bottleneck? (add cache / read replicas / sharding)
[ ] Single points of failure? (add redundancy everywhere)
[ ] Data consistency model? (strong vs eventual — justify the choice)
[ ] How does the system behave when a component fails?
[ ] How to handle hot keys / celebrity problem?
```

---

## Red Flags to Avoid

| Anti-pattern | What to Do Instead |
|-------------|-------------------|
| Starting to draw before clarifying | Ask requirements first |
| Over-engineering in the first 5 min | Start simple, add complexity when asked |
| Saying "use a database" | Name the specific DB and explain why |
| Ignoring failure scenarios | Always mention replication and fallbacks |
| Silent designing | Think out loud, narrate your decisions |
| Tunnel vision on one component | Keep the big picture visible |

---

## Phrases That Impress Interviewers

- "I'd choose X here because it gives us Y, with the trade-off of Z."
- "This is a read-heavy system, so I'll optimise the read path first."
- "Before I go deeper on this, should I also cover [other component]?"
- "This is a good place to introduce a cache — here's why..."
- "We can start with a single region and expand globally by adding..."
- "The bottleneck here is [X]. I'd address it by [Y]."

---

## Sample Answer Structure (30-minute interview)

```
0:00 - 5:00   Ask clarifying questions, confirm scope
5:00 - 8:00   Estimate scale (DAU, QPS, storage)
8:00 - 15:00  Draw high-level architecture, explain each box
15:00 - 25:00 Deep dive on 2-3 components (as directed by interviewer)
25:00 - 30:00 Talk through scaling, failure handling, trade-offs
```
