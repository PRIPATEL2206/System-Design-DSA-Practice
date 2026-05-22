# Greedy & Backtracking — Optimization and Exhaustive Search

## Greedy Algorithms

### Core Idea
At each step, make the **locally optimal choice** hoping it leads to the globally optimal solution. No looking back, no reconsidering.

### When Greedy Works
- Problem has **greedy choice property** (local optimal → global optimal)
- Problem has **optimal substructure**
- You can prove the greedy choice is safe (or it's a known greedy problem)

### When Greedy FAILS
If making the best local choice can trap you in a suboptimal global solution, you need DP or backtracking instead.

```
Example: Coin change with denominations [1, 3, 4], target = 6
Greedy: 4 + 1 + 1 = 3 coins
Optimal: 3 + 3 = 2 coins ← Greedy fails!
```

### Essential Greedy Patterns

**Pattern 1: Interval Scheduling (Meeting Rooms, Merge Intervals)**
```python
# Merge Intervals (asked at every FAANG)
def merge(intervals):
    intervals.sort(key=lambda x: x[0])
    merged = [intervals[0]]
    for start, end in intervals[1:]:
        if start <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], end)
        else:
            merged.append([start, end])
    return merged

# Non-overlapping Intervals (minimum removals)
def erase_overlap(intervals):
    intervals.sort(key=lambda x: x[1])  # Sort by END time
    count = 0
    prev_end = float('-inf')
    for start, end in intervals:
        if start >= prev_end:
            prev_end = end
        else:
            count += 1
    return count

# Meeting Rooms II (minimum rooms needed)
def min_meeting_rooms(intervals):
    starts = sorted(i[0] for i in intervals)
    ends = sorted(i[1] for i in intervals)
    rooms = 0
    max_rooms = 0
    s, e = 0, 0
    while s < len(starts):
        if starts[s] < ends[e]:
            rooms += 1
            max_rooms = max(max_rooms, rooms)
            s += 1
        else:
            rooms -= 1
            e += 1
    return max_rooms
```

**Pattern 2: Activity Selection / Job Scheduling**
```python
# Maximum number of non-overlapping activities
def max_activities(activities):
    activities.sort(key=lambda x: x[1])  # Sort by finish time
    count = 1
    last_end = activities[0][1]
    for start, end in activities[1:]:
        if start >= last_end:
            count += 1
            last_end = end
    return count
```

**Pattern 3: Jump Game**
```python
# Can you reach the last index?
def can_jump(nums):
    max_reach = 0
    for i in range(len(nums)):
        if i > max_reach:
            return False
        max_reach = max(max_reach, i + nums[i])
    return True

# Minimum jumps to reach end
def min_jumps(nums):
    jumps = 0
    curr_end = 0
    farthest = 0
    for i in range(len(nums) - 1):
        farthest = max(farthest, i + nums[i])
        if i == curr_end:
            jumps += 1
            curr_end = farthest
    return jumps
```

**Pattern 4: Gas Station**
```python
def can_complete_circuit(gas, cost):
    if sum(gas) < sum(cost):
        return -1
    tank = 0
    start = 0
    for i in range(len(gas)):
        tank += gas[i] - cost[i]
        if tank < 0:
            tank = 0
            start = i + 1
    return start
```

---

## Backtracking

### Core Idea
Explore ALL possible solutions by building candidates incrementally and **abandoning (backtracking)** a candidate as soon as you determine it can't lead to a valid solution.

### Template
```python
def backtrack(state, choices):
    if is_solution(state):
        result.append(state.copy())
        return
    
    for choice in choices:
        if is_valid(choice, state):
            make_choice(state, choice)
            backtrack(state, remaining_choices)
            undo_choice(state, choice)  # BACKTRACK
```

### Essential Backtracking Problems

**Pattern 5: Subsets (Power Set)**
```python
def subsets(nums):
    result = []
    def backtrack(start, current):
        result.append(current[:])
        for i in range(start, len(nums)):
            current.append(nums[i])
            backtrack(i + 1, current)
            current.pop()
    backtrack(0, [])
    return result

# With duplicates
def subsets_with_dup(nums):
    nums.sort()
    result = []
    def backtrack(start, current):
        result.append(current[:])
        for i in range(start, len(nums)):
            if i > start and nums[i] == nums[i-1]:
                continue  # Skip duplicates
            current.append(nums[i])
            backtrack(i + 1, current)
            current.pop()
    backtrack(0, [])
    return result
```

**Pattern 6: Permutations**
```python
def permutations(nums):
    result = []
    def backtrack(current, remaining):
        if not remaining:
            result.append(current[:])
            return
        for i in range(len(remaining)):
            current.append(remaining[i])
            backtrack(current, remaining[:i] + remaining[i+1:])
            current.pop()
    backtrack([], nums)
    return result
```

**Pattern 7: Combination Sum**
```python
# Elements can be used unlimited times
def combination_sum(candidates, target):
    result = []
    def backtrack(start, current, remaining):
        if remaining == 0:
            result.append(current[:])
            return
        for i in range(start, len(candidates)):
            if candidates[i] > remaining:
                break
            current.append(candidates[i])
            backtrack(i, current, remaining - candidates[i])  # i, not i+1 (reuse)
            current.pop()
    candidates.sort()
    backtrack(0, [], target)
    return result
```

**Pattern 8: N-Queens**
```python
def solve_n_queens(n):
    result = []
    cols = set()
    diag1 = set()  # row - col
    diag2 = set()  # row + col
    board = [['.' ] * n for _ in range(n)]
    
    def backtrack(row):
        if row == n:
            result.append([''.join(r) for r in board])
            return
        for col in range(n):
            if col in cols or (row-col) in diag1 or (row+col) in diag2:
                continue
            cols.add(col)
            diag1.add(row - col)
            diag2.add(row + col)
            board[row][col] = 'Q'
            backtrack(row + 1)
            board[row][col] = '.'
            cols.remove(col)
            diag1.remove(row - col)
            diag2.remove(row + col)
    
    backtrack(0)
    return result
```

**Pattern 9: Word Search in Grid**
```python
def exist(board, word):
    rows, cols = len(board), len(board[0])
    
    def backtrack(r, c, idx):
        if idx == len(word):
            return True
        if (r < 0 or r >= rows or c < 0 or c >= cols or 
            board[r][c] != word[idx]):
            return False
        
        temp = board[r][c]
        board[r][c] = '#'  # Mark visited
        
        found = (backtrack(r+1, c, idx+1) or backtrack(r-1, c, idx+1) or
                 backtrack(r, c+1, idx+1) or backtrack(r, c-1, idx+1))
        
        board[r][c] = temp  # Restore
        return found
    
    for r in range(rows):
        for c in range(cols):
            if backtrack(r, c, 0):
                return True
    return False
```

**Pattern 10: Palindrome Partitioning**
```python
def partition(s):
    result = []
    def backtrack(start, current):
        if start == len(s):
            result.append(current[:])
            return
        for end in range(start + 1, len(s) + 1):
            substring = s[start:end]
            if substring == substring[::-1]:  # Is palindrome
                current.append(substring)
                backtrack(end, current)
                current.pop()
    backtrack(0, [])
    return result
```

---

## Greedy vs DP vs Backtracking — Decision Guide

| Signal | Approach |
|--------|----------|
| "Maximum/minimum" + clear local choice | Try Greedy first |
| "Maximum/minimum" + overlapping subproblems | DP |
| "Find ALL solutions" / "Count all ways" | Backtracking (or DP for count) |
| "Can you partition/subset..." | DP (subset sum) or Backtracking |
| Small n (≤ 20) + "all combinations" | Backtracking |
| Intervals + ordering | Greedy (sort by end) |

---

## Must-Solve Problems

### Greedy
| Problem | Key Insight |
|---------|-------------|
| Merge Intervals | Sort by start, extend or add |
| Non-overlapping Intervals | Sort by end time |
| Jump Game I & II | Track farthest reach |
| Gas Station | Reset if tank goes negative |
| Meeting Rooms II | Track start/end events |
| Assign Cookies | Sort both, match smallest first |

### Backtracking
| Problem | Template Variation |
|---------|-------------------|
| Subsets | Include/exclude each element |
| Permutations | Choose from remaining |
| Combination Sum | Reuse allowed, start from i |
| N-Queens | Place row by row, check diagonals |
| Word Search | Grid DFS with backtrack |
| Palindrome Partitioning | Partition at palindrome boundaries |
| Generate Parentheses | Track open/close count |
| Letter Combinations of Phone | Map digits to letters |

---

## Common Mistakes

1. **Applying greedy without proof** — Not every optimization problem is greedy. If unsure, use DP.
2. **Forgetting to undo choices in backtracking** — The "pop" or "restore" step is essential.
3. **Not pruning early** — Sort candidates and break early when sum exceeds target.
4. **Duplicate handling** — Sort + skip `if i > start and nums[i] == nums[i-1]`.
5. **Confusing combinations vs permutations** — Combinations use `start` parameter; permutations don't.
