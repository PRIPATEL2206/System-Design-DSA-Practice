# DSA Interview Playbook — Tactics That Win Offers

## The Truth About DSA Interviews

It's NOT just about solving the problem. Google's internal hiring docs reveal the scoring:
- **Problem Solving (40%):** Approach, optimization, pattern recognition
- **Coding (30%):** Clean, correct, efficient implementation
- **Communication (20%):** Thinking out loud, responding to hints
- **Verification (10%):** Testing, edge cases

You can get an offer without solving every problem perfectly — IF your process is strong.

---

## Before the Interview

### Mental Preparation
- Sleep 7+ hours the night before
- Eat a good meal 1-2 hours before
- Have water nearby
- Accept: "I might get stuck, and that's OK. I have a recovery plan."

### Technical Preparation (Last Week)
- Review your top 3 weak patterns
- Solve 2-3 medium problems per day (don't grind new hards)
- Practice explaining solutions out loud
- Time yourself: 25 min for medium, 35 min for hard

---

## During the Interview — Minute by Minute

### Minutes 0-2: Greeting
- Be warm, professional
- "Hi, I'm Prince. Great to meet you."
- Brief small talk if interviewer initiates

### Minutes 2-7: Understanding the Problem
```
1. Read/listen carefully
2. Restate: "So the goal is to... given these constraints..."
3. Ask clarifying questions:
   - Input size? (Tells you target complexity)
   - Can there be negative numbers / empty input?
   - Is the input sorted?
   - Should I optimize for time or space?
4. Write 2-3 examples (including edge case)
5. Confirm: "Does this look right before I discuss my approach?"
```

### Minutes 7-12: Approach Discussion
```
1. Start with brute force: "The naive approach would be..."
2. State its complexity: "That's O(n²) which might be too slow for n=10⁵"
3. Optimize: "I notice that [insight]... If I use [technique], I can get O(n)"
4. State the plan clearly: "So my approach is:
   - Build a hash map of values to indices
   - For each element, check if complement exists
   - Return indices when found"
5. Confirm: "Shall I code this up?"
```

### Minutes 12-30: Coding
```
- Talk while coding: "I'm initializing my hash map here..."
- Use meaningful names (not 'i', 'j', 'temp' — use 'left', 'right', 'current_sum')
- Handle edge cases first: if not nums: return []
- Write clean code (helper functions if needed)
- If you make a mistake, say "Actually, let me fix that" (don't panic)
```

### Minutes 30-37: Verification
```
1. "Let me trace through my example..."
2. Walk through your code with the example (simulate execution)
3. Check edge cases mentally
4. State complexity: "This is O(n) time, O(n) space"
```

### Minutes 37-45: Follow-up
The interviewer might ask:
- "Can you optimize space?" → Think in-place techniques
- "What if the array is very large?" → Streaming / external sort
- "What if we need thread safety?" → Discuss locking or CAS
- "How would you test this?" → Unit tests, edge cases, stress tests

---

## How to Handle Specific Situations

### "I'm Stuck" (The 30-Second Rule)
If you've been thinking for 30 seconds without progress:
```
SAY: "I'm going to take a step back. Let me think about what data structure 
     would help here... [pause 10 sec]... I think a hash map might work 
     because I need O(1) lookups for the complement."
```

If still stuck after 1 minute:
```
SAY: "I have a couple of thoughts but I'm not sure which direction is better. 
     Could I get a small hint about which approach might be more fruitful?"
```

### "I Made a Bug"
Don't panic. Bugs during coding are EXPECTED.
```
SAY: "Ah, I see the issue — my loop bound should be len(arr) - 1, not len(arr), 
     because I'm comparing with arr[i+1]. Let me fix that."
```

### "I Don't Know This Algorithm"
```
SAY: "I haven't seen this exact pattern before, but let me think about it from 
     first principles. The key constraint seems to be [X], which suggests 
     [approach]. Let me try building from there."
```

### "The Problem is Too Easy"
Solve it quickly and cleanly. Then proactively optimize:
```
SAY: "I've solved it in O(n) time and O(n) space. If we wanted O(1) space, 
     we could [alternative approach]. Would you like me to implement that?"
```

---

## The Interviewer's Perspective

### What Gets You "Strong Hire"
- Identified the correct approach within 5 minutes
- Coded cleanly with minimal bugs
- Handled edge cases without being prompted
- Explained trade-offs proactively
- Responded well to follow-up questions

### What Gets You "Hire"
- Needed one hint but recovered quickly
- Clean code with 1-2 bugs that you found yourself
- Good communication throughout
- Solid complexity analysis

### What Gets You "No Hire"
- Couldn't identify approach even with hints
- Code was messy and hard to follow
- Silent thinking for extended periods
- Gave up or became frustrated
- Couldn't analyze complexity

---

## Common Patterns by Company

### Google
- Heavy on graphs, dynamic programming, and strings
- Often asks "What's the optimal solution?" after you code something
- Values clean code and mathematical thinking
- May ask you to write unit tests

### Amazon
- Loves arrays (merge intervals, meeting rooms, stock problems)
- Often ties to leadership principles: "Tell me about the trade-off"
- Practical problems: design a cache, implement a file system
- Values working code over perfect code

### Meta (Facebook)
- Sliding window, prefix sum, and tree problems dominate
- Fast pace — may ask 2 problems in 45 minutes
- Values speed + correctness
- Often tests BFS/DFS variations

### Microsoft
- More forgiving — expects structured thinking even if you don't finish
- Mix of easy + medium problems
- Often starts with an easy warm-up, then medium follow-up
- Values communication and learning ability

### Apple
- String manipulation, array problems, linked lists
- Clean code is highly valued
- May ask follow-up about how to test your solution
- Values practical engineering judgment

---

## Practice Strategy

### The 4-Week Sprint Plan
```
Week 1: Easy problems (2/day) — build confidence and speed
Week 2: Medium problems (2/day) — pattern recognition
Week 3: Medium-Hard (1-2/day) — optimization and recovery skills
Week 4: Mock interviews + review — simulate real conditions
```

### How to Practice Effectively
1. **Set a timer** (25 min for medium, 40 for hard)
2. **Don't look at hints** until timer expires
3. **After solving:** Read discussion, learn alternative approaches
4. **After failing:** Understand the pattern, solve 2 similar problems
5. **Weekly:** Do 1 mock interview (Pramp, interviewing.io, or with a friend)

### The "Pattern Drill" Method
Instead of random LeetCode:
1. Pick one pattern (e.g., sliding window)
2. Solve 5 problems using that pattern
3. After solving, identify the "trigger" that tells you it's this pattern
4. Move to next pattern
5. After all patterns: do mixed practice
