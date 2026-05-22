# Complete Interview Playbook — Strategy That Works

## The Big Picture

```
Timeline (10 weeks):
────────────────────
Week 1-2:   Foundation (CS fundamentals + resume)
Week 3-6:   Deep Practice (DSA + System Design daily)
Week 7-8:   Mock Interviews + Behavioral prep
Week 9:     Company-specific prep + applications
Week 10:    Light revision + cheatsheets + rest

Daily Schedule (3-4 hours):
───────────────────────────
Morning (1.5h):  DSA — 2-3 problems (focused, timed)
Afternoon (1h):  System Design OR CS fundamentals
Evening (1h):    Behavioral stories OR mock interview
```

---

## 1. Interview Process at Big Tech

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TYPICAL FAANG INTERVIEW PROCESS                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. RESUME SCREEN (recruiter reviews)                               │
│     → Pass rate: ~5-10%                                             │
│     → Timeline: 1-2 weeks                                           │
│                                                                      │
│  2. PHONE SCREEN / ONLINE ASSESSMENT                                │
│     → 1-2 coding problems (45 min)                                  │
│     → Medium difficulty, must solve both                            │
│     → Pass rate: ~30-40%                                            │
│                                                                      │
│  3. ONSITE (Virtual or In-person) — 4-6 rounds                     │
│     → 2-3 Coding rounds (DSA)                                       │
│     → 1 System Design (senior level)                                │
│     → 1-2 Behavioral / Culture fit                                  │
│     → Sometimes: Domain-specific (ML, frontend, etc.)               │
│     → Pass rate: ~20-30%                                            │
│                                                                      │
│  4. HIRING COMMITTEE REVIEW                                          │
│     → Reviews all feedback                                           │
│     → May request additional interviews                             │
│                                                                      │
│  5. TEAM MATCHING + OFFER                                           │
│     → Negotiate! (see salary negotiation guide)                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Coding Interview Strategy

### Before the Interview
```
1. Practice on LeetCode/Codeforces DAILY (minimum 2 problems)
2. Focus on patterns, not memorizing solutions
3. Time yourself: Easy < 10 min, Medium < 25 min, Hard < 40 min
4. Practice on whiteboard/Google Docs (not just IDE!)
5. Explain your thinking OUT LOUD while solving
```

### During the Interview (45 minutes)
```
[0-3 min] UNDERSTAND the problem
────────────────────────────────
- Repeat the problem in your own words
- Ask clarifying questions:
  "What's the input size?"
  "Can there be duplicates?"
  "What should I return if input is empty?"
  "Is the array sorted?"
- Write down 2-3 examples (including edge cases)

[3-8 min] PLAN your approach
─────────────────────────────
- State the brute force first: "The naive approach would be O(n²)..."
- Identify the pattern (sliding window? two pointer? BFS?)
- Propose your optimized approach
- State time/space complexity BEFORE coding
- Get interviewer's buy-in: "Does this approach sound good?"

[8-35 min] CODE the solution
─────────────────────────────
- Write clean, readable code
- Use meaningful variable names (not i, j, k everywhere)
- Handle edge cases first (empty input, single element)
- Talk through your logic as you write
- Don't get stuck on syntax — pseudocode is OK initially

[35-42 min] TEST your solution
──────────────────────────────
- Walk through with your examples manually
- Test edge cases: empty, single element, all same, maximum size
- Fix any bugs you find

[42-45 min] OPTIMIZE (if time)
──────────────────────────────
- Can you reduce space complexity?
- Any potential issues at scale?
- Alternative approaches?
```

### What Interviewers Actually Evaluate
```
┌─────────────────────────────────────────────────────────────┐
│ Criteria          │ Weight │ How to Score High               │
├───────────────────┼────────┼─────────────────────────────────┤
│ Problem Solving   │ 35%    │ Identify pattern quickly,       │
│                   │        │ handle edge cases               │
│ Code Quality      │ 25%    │ Clean, readable, modular        │
│ Communication     │ 20%    │ Think aloud, explain trade-offs │
│ Testing           │ 10%    │ Walk through examples, find bugs│
│ Optimization      │ 10%    │ Know time/space complexity      │
└───────────────────┴────────┴─────────────────────────────────┘

You CAN get hired without solving perfectly!
A well-communicated 80% solution beats a silent 100% solution.
```

---

## 3. System Design Strategy

```
[0-5 min]   CLARIFY requirements
[5-10 min]  ESTIMATE scale (back-of-envelope)
[10-15 min] HIGH-LEVEL design (boxes and arrows)
[15-35 min] DEEP DIVE (data model, APIs, scaling)
[35-40 min] BOTTLENECKS & trade-offs
[40-45 min] WRAP-UP & extensions

Key phrases that score points:
- "The bottleneck here is..."
- "The trade-off between X and Y..."
- "At this scale, we'd need..."
- "For consistency, I'd choose... because..."
- "If this fails, the fallback is..."
```

---

## 4. Behavioral Interview Strategy

### STAR Method (Structure EVERY Answer)
```
S — Situation: Context (1-2 sentences)
T — Task: What was YOUR responsibility
A — Action: What YOU did (specific steps)
R — Result: Quantifiable outcome

Example:
Q: "Tell me about a time you dealt with a difficult deadline"

S: "At TCS, we had a data pipeline migration that was originally scoped 
    for 8 weeks, but client moved the deadline to 4 weeks."
T: "I was responsible for the ETL pipeline redesign and team of 3."
A: "I broke the work into phases, identified we could parallelize the 
    schema migration and testing. I set up daily standups, automated 
    testing early, and deprioritized non-critical features for v2."
R: "We delivered 2 days early with zero data loss. The client extended 
    our contract by 6 months based on this delivery."
```

### Top 10 Stories You MUST Prepare
```
1. Biggest technical challenge you solved
2. Time you disagreed with a teammate/manager
3. Project you're most proud of
4. Time you failed and what you learned
5. Time you had to learn something quickly
6. Time you mentored someone / showed leadership
7. Time you improved a process or system
8. Time you dealt with ambiguity / unclear requirements
9. Time you had to make a decision with incomplete data
10. Why this company? Why this role? Why now?
```

---

## 5. Application Strategy

### Where to Apply (Priority Order)
```
Tier 1 (Dream companies): Google, Meta, Amazon, Microsoft, Apple
Tier 2 (Excellent companies): Netflix, Uber, Stripe, Airbnb, LinkedIn
Tier 3 (Great companies): Atlassian, Salesforce, Adobe, VMware, Intuit
Tier 4 (Practice interviews): Smaller companies for real practice

Strategy:
- Apply to Tier 4 FIRST (get interview practice)
- Apply to Tier 2-3 NEXT (build confidence)
- Apply to Tier 1 LAST (when you're at peak performance)
- Apply to 20-30 companies total (numbers game)
```

### Getting Referrals (10x better than cold apply)
```
1. LinkedIn: Connect with employees, ask for coffee chat
2. Alumni network: Same college → instant connection
3. Tech meetups: Build relationships before asking
4. Open source: Contribute to their repos → natural conversation
5. Direct message: "Hi, I'm interested in [role]. Could I ask you 
   3 questions about your experience at [company]?"

Referral email template:
"Hi [Name], I'm a [your role] with [X years] experience in [tech]. 
I'm very interested in the [specific role] at [company] because 
[specific reason]. Would you be open to referring me? I've attached 
my resume. Happy to chat about my background if helpful."
```

---

## 6. Company-Specific Tips

### Google
```
Focus: DSA (hard), System Design, Googleyness (culture)
Unique: Hiring committee decides (not interviewer). Need 3+ "hire" votes.
Tips: 
- Solve problems optimally (they care about complexity)
- Explain trade-offs clearly
- Show curiosity and ability to go deep
- "Googleyness" = intellectual humility + collaboration
```

### Amazon
```
Focus: Leadership Principles (14 LPs), DSA, System Design
Unique: Every behavioral question maps to an LP. Prepare 2 stories per LP.
Key LPs:
- Customer Obsession (always start with customer)
- Ownership (don't say "that's not my job")
- Bias for Action (speed matters)
- Dive Deep (know the details)
- Deliver Results (quantify EVERYTHING)
```

### Meta
```
Focus: DSA (move fast!), System Design, Behavioral (values)
Unique: Speed matters! Solve 2 problems in 45 min. They value "move fast."
Tips:
- Practice solving medium problems in 15-20 min
- System design: focus on scale (Meta scale = billions)
- Show you can build AND ship quickly
```

### Microsoft
```
Focus: DSA (moderate difficulty), System Design, Collaboration
Unique: More collaborative interview style. Interviewers help more.
Tips:
- Focus on communication and problem-solving process
- They value growth mindset
- Demonstrate ability to work in teams
- Ask great questions about the team/project
```

### Apple
```
Focus: Domain expertise, DSA, Design thinking
Unique: More secretive. May not tell you which team until offer.
Tips:
- Deep expertise in your area matters more
- Show passion for great products
- Privacy and security awareness appreciated
- Less "standard" interview — more conversation
```

---

## 7. During Interview Day

```
Before:
  □ Good night's sleep (8 hours minimum)
  □ Light breakfast (protein, not heavy carbs)
  □ Review cheatsheet (30 min max — don't cram)
  □ Test setup: camera, mic, IDE, stable internet
  □ Have water and notepad ready
  □ Open IDE/whiteboard tool, test it works

During:
  □ Smile, be energetic (they're evaluating "would I work with this person?")
  □ Take 10 seconds to think before speaking (silence is OK!)
  □ Ask clarifying questions (shows maturity)
  □ Think aloud (they can't read your mind)
  □ If stuck: "Let me think about this differently..."
  □ Manage time (don't spend 20 min on understanding)

After each round:
  □ 5-minute break: water, stretch, reset
  □ Don't dwell on previous round (fresh start each time)
  □ Note any questions you want to ask in next round

After interview:
  □ Send thank-you email within 24 hours
  □ Note what went well and what to improve
  □ Start preparing for next company (don't stop momentum)
```
