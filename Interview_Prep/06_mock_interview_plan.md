# Mock Interview Plan — Day-by-Day Schedule

## Pre-Interview Checklist

- [ ] Can explain every line on your resume confidently
- [ ] Can draw ETL pipeline architecture from memory
- [ ] Can code LRU Cache, Rate Limiter from scratch
- [ ] Can design a system end-to-end in 25 minutes
- [ ] Can answer "Tell me about yourself" in 90 seconds
- [ ] Have 3 failure stories ready (STAR format)
- [ ] Have 2 "What would you do differently" answers ready

---

## If You Have 7 Days

| Day | Morning (2 hrs) | Evening (2 hrs) |
|-----|-----------------|-----------------|
| **Day 1** | Study: System Design fundamentals (file 02) | Practice: Design ETL pipeline on paper |
| **Day 2** | Study: LLD patterns + SOLID (file 03, Part C) | Practice: Code Rate Limiter + LRU Cache |
| **Day 3** | Study: AWS Deep-dive (file 05, Section A) | Practice: Design Real-time Chat System |
| **Day 4** | Study: PySpark + SQL optimization (file 05, Section B) | Practice: Code Task Scheduler + Connection Pool |
| **Day 5** | Study: Project deep-dive prep (file 04) | Practice: Mock interview with friend (HLD) |
| **Day 6** | Study: ML System Design (file 01, Section 6) | Practice: Mock interview with friend (LLD + coding) |
| **Day 7** | Revision: Cheatsheets + weak areas | Light practice: Re-draw 2 designs, relax |

---

## If You Have 3 Days (Crash Course)

| Day | Focus | Key Actions |
|-----|-------|-------------|
| **Day 1** | Design + Architecture | Read files 02 + 05. Draw ETL pipeline + Chat System designs. |
| **Day 2** | Implementation + Coding | Read file 03. Code: LRU Cache, Rate Limiter, Circuit Breaker. |
| **Day 3** | Project stories + Revision | Read file 04. Practice STAR answers. Review all cheatsheets. |

---

## If You Have 1 Day (Emergency)

**Morning:**
1. Read file 04 (project deep-dive) — you MUST know your own projects
2. Practice explaining ETL architecture in 5 minutes
3. Code one LLD problem (LRU Cache or Rate Limiter)

**Afternoon:**
1. Read System Design template (file 02, bottom section)
2. Practice one full design: "Design a Data Pipeline" (25 min timer)
3. Review AWS optimization answers (file 05, Q1-Q5)

**Evening:**
1. Read cheatsheets: `opus/career_guide/06_cheatsheets/`
2. Prepare "Tell me about yourself" (90 sec version)
3. Sleep well

---

## Mock Interview Script (Practice with a Friend or Solo)

### Round Simulation (45 minutes total)

**Minutes 0-5: Introduction**
- "Tell me about yourself" (90 seconds)
- "Walk me through your current project at TCS" (3 minutes)

**Minutes 5-25: System Design**
- "Design a system that processes 100M healthcare records daily"
- Start: Requirements → Estimate → Architecture → Deep-dive → Trade-offs

**Minutes 25-40: Implementation**
- "Implement a rate limiter / LRU cache / workflow engine"
- Code on paper/whiteboard (no IDE autocomplete)
- Explain as you code

**Minutes 40-45: Questions**
- "What questions do you have for us?"
- Prepare 2-3 thoughtful questions about the team/tech

---

## Self-Evaluation Rubric

After each practice session, rate yourself:

| Criteria | Score (1-5) | Notes |
|----------|-------------|-------|
| Clarified requirements before starting | | |
| Structured approach (not random) | | |
| Communicated trade-offs | | |
| Code was clean and working | | |
| Handled edge cases | | |
| Related to real experience | | |
| Stayed within time limit | | |
| Confident delivery (no rambling) | | |

**Target: 4+ in each category before interview day.**

---

## Power Phrases to Use in Interview

| Situation | Say This |
|-----------|----------|
| Starting a design | "Let me start by clarifying the requirements and scale..." |
| Making a choice | "I'd choose X over Y because..., the trade-off is..." |
| Relating to experience | "In my TCS project, we faced a similar challenge where..." |
| Handling unknowns | "I'm not sure about the exact implementation, but my approach would be..." |
| Finishing | "To summarize, the key design decisions are... and the main trade-offs are..." |

---

## Common Mistakes to Avoid

1. **Jumping to solution** without asking questions
2. **Over-engineering** — keep it simple first, then scale
3. **Ignoring non-functional requirements** (latency, cost, security)
4. **Not relating to experience** — they hired you for your background
5. **Silent coding** — always explain your thought process
6. **Perfectionism** — working code > perfect code in interviews
7. **Forgetting monitoring** — always mention how you'd know it's broken
