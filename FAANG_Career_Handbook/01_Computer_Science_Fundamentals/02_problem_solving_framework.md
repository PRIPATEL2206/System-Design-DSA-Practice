# Problem-Solving Framework — How to Think in Interviews

## The #1 Reason People Fail Coding Interviews

It's NOT that they don't know the algorithm. It's that they **panic, jump to code, and get stuck** because they never built a systematic approach to breaking down problems.

This framework is your safety net. Follow it every time.

---

## The 5-Step FAANG Problem-Solving Framework

### Step 1: UNDERSTAND (2-3 minutes)

Before touching code, make sure you truly understand the problem.

**Do this:**
- Restate the problem in your own words to the interviewer
- Identify inputs and outputs precisely (types, ranges, edge cases)
- Ask clarifying questions

**Clarifying questions to ALWAYS ask:**
```
- What's the input size range? (This hints at required complexity)
- Can there be duplicates?
- Is the input sorted?
- Can values be negative?
- What should I return if input is empty?
- Are there constraints on space?
- Should I optimize for time or space?
```

**Why this matters:** At Google, asking good questions IS part of the evaluation. They want to see you think before you code.

### Step 2: EXAMPLES (2-3 minutes)

Work through 2-3 examples BY HAND on the whiteboard/screen.

**Always include:**
1. A normal case
2. An edge case (empty, single element, all same values)
3. A case that tests your assumptions

**Example — "Find two numbers that sum to target":**
```
Input: [2, 7, 11, 15], target = 9
→ Walk through: 2+7=9 ✓ → return [0, 1]

Edge: [3, 3], target = 6
→ Same number appears twice → return [0, 1]

Edge: [], target = 5
→ Empty array → return []
```

**The magic:** Working examples by hand often reveals the algorithm naturally.

### Step 3: APPROACH (3-5 minutes)

Think out loud. Start brute force, then optimize.

**The optimization ladder:**
```
1. Brute force → What's the dumbest thing that works?
2. Identify waste → What are you computing repeatedly?
3. Apply technique → Can a data structure eliminate that waste?
4. Verify → Does this handle all edge cases?
```

**Communicate like this:**
> "The brute force would be O(n²) using two nested loops to check all pairs. But I notice we're repeatedly searching for a complement... If I use a hash map to store values I've seen, I can find the complement in O(1), making it O(n) overall."

**Pattern matching (jump to Step 7 in 07_pattern_recognition_guide.md):**
- Sorted array? → Two pointers or binary search
- Subarray/substring? → Sliding window
- Optimal substructure + overlapping subproblems? → DP
- "All combinations/permutations"? → Backtracking
- "Top K" or "Kth largest"? → Heap
- Graph traversal? → BFS (shortest) or DFS (explore all)

### Step 4: CODE (10-15 minutes)

Now and only now, start writing code.

**Rules for interview coding:**
- Write clean, readable code (meaningful variable names)
- Talk through what you're writing
- Don't optimize prematurely — get a working solution first
- Use helper functions for clarity
- Leave space for edge case handling at the top

**Template:**
```python
def solve(input_params):
    # Edge cases
    if not input_params:
        return default_value
    
    # Initialize data structures
    seen = {}
    
    # Main logic
    for item in input_params:
        # Core algorithm
        pass
    
    return result
```

### Step 5: VERIFY (3-5 minutes)

Don't say "I think this works." PROVE it works.

**Verification checklist:**
- [ ] Trace through your normal example
- [ ] Trace through your edge cases
- [ ] Check off-by-one errors (loop bounds, indices)
- [ ] Check that you handle the empty/null case
- [ ] State the final time and space complexity

---

## How to Get Unstuck (The Recovery Playbook)

Getting stuck happens. What separates FAANG hires from rejects is **how they recover**.

### Technique 1: Simplify the Problem
Can't solve for n? Solve for n=1, n=2, n=3. See the pattern emerge.

### Technique 2: Think About Data Structures
"What data structure gives me the operation I need in O(1)?"
- Need fast lookup? → Hash map
- Need ordering? → Sorted set / BST
- Need min/max quickly? → Heap
- Need LIFO? → Stack
- Need FIFO? → Queue

### Technique 3: Work Backwards
"If I had the answer, how would I verify it?" Sometimes verification logic IS the algorithm (e.g., binary search on answer).

### Technique 4: Draw It Out
For trees, graphs, and linked lists — ALWAYS draw. Visualizing pointers and connections reveals the algorithm.

### Technique 5: Relate to Known Problems
"This reminds me of..." — Most interview problems are variations of ~15 core patterns.

### What to Say When Stuck
```
"I'm going to take a moment to think about this differently..."
"Let me consider what data structure would help here..."
"I see this has overlap with [pattern] — let me think about whether that applies..."
"Can I ask — is it guaranteed that [constraint]? That would change my approach."
```

**NEVER:**
- Sit in silence for > 30 seconds
- Say "I don't know"
- Give up

---

## Thinking Out Loud — The FAANG Difference

At FAANG companies, your thought process is worth 40-50% of the evaluation. A candidate who explains their approach clearly but codes slowly will often beat someone who codes fast but can't explain why.

### What "thinking out loud" sounds like:

**Bad:** *silence* ... *writes code* ... "done"

**Good:**
> "So I need to find the longest substring without repeating characters. Let me think about what approaches could work here..."
> 
> "If I use brute force, I'd check every possible substring — that's O(n³) with the substring generation and checking. Too slow for n=10⁵."
>
> "I notice that as I scan left to right, I only need to 'forget' characters when I see a repeat. This feels like a sliding window — I can expand right, and shrink from left when I hit a duplicate."
>
> "I'll use a hash set to track characters in my current window, and two pointers for the window boundaries. Let me code this up..."

---

## Time Management in a 45-Minute Interview

| Phase | Time | What to do |
|-------|------|-----------|
| Understand + Examples | 5 min | Ask questions, write examples |
| Approach | 5 min | Think out loud, state approach |
| Code | 15-20 min | Write clean solution |
| Verify + Optimize | 5-10 min | Trace through, fix bugs |
| Follow-up | 5-10 min | Handle interviewer's extensions |

**Key insight:** If you haven't started coding by minute 12, you're behind. It's better to code a brute force and optimize than to think for 20 minutes about the perfect solution.

---

## Common Interview Mistakes

| Mistake | Fix |
|---------|-----|
| Jumping straight to code | Force yourself: examples FIRST |
| Over-engineering | Start simple, optimize after |
| Not testing edge cases | Always trace empty/single/duplicate |
| Silent thinking | Narrate your thought process |
| Giving up when stuck | Use recovery techniques above |
| Ignoring hints | Interviewer hints are GIFTS — use them |
| Not asking input constraints | This tells you the required complexity |

---

## Practice This Framework

Take any LeetCode medium problem and force yourself through all 5 steps, timing each phase. After 20 problems, the framework becomes automatic.

**The goal:** When you sit in a Google interview, you don't think "What do I do?" — your body already knows the flow: understand → example → approach → code → verify.
