# Complexity Analysis — The Language of Efficient Code

## Why This Matters for Interviews

Every single coding interview evaluates your solution on two axes: **correctness** and **efficiency**. You will be asked "What's the time and space complexity?" after every solution. If you can't answer clearly, you won't pass.

At FAANG, a brute force solution is only the starting point. They want to see you optimize.

---

## Big-O Notation — The Core Idea

Big-O describes **how your algorithm scales** as input grows. It's not about exact time — it's about the growth rate.

### The Mental Model

Think of it as: "If I double my input, what happens to my runtime?"

| Complexity | Name | Doubling Input Effect | Example |
|-----------|------|----------------------|---------|
| O(1) | Constant | No change | Hash table lookup |
| O(log n) | Logarithmic | +1 operation | Binary search |
| O(n) | Linear | 2x time | Single loop |
| O(n log n) | Linearithmic | Slightly > 2x | Merge sort |
| O(n²) | Quadratic | 4x time | Nested loops |
| O(2ⁿ) | Exponential | Impossibly worse | All subsets |
| O(n!) | Factorial | Universe ends | All permutations |

### The Intuition Behind Each

**O(1) — Constant:**
You go directly to what you need. Like looking up a word in a dictionary by page number (not searching).
```python
def get_first(arr):
    return arr[0]  # Always 1 operation, regardless of array size
```

**O(log n) — Logarithmic:**
You eliminate half the remaining work each step. Like binary search — 1 billion elements? Only 30 steps.
```python
def binary_search(arr, target):
    lo, hi = 0, len(arr) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1
```

**O(n) — Linear:**
You touch every element exactly once. If input doubles, time doubles.
```python
def find_max(arr):
    max_val = arr[0]
    for num in arr:
        if num > max_val:
            max_val = num
    return max_val
```

**O(n log n) — Linearithmic:**
You do something logarithmic for each element. This is the **speed limit** for comparison-based sorting.
```python
arr.sort()  # Timsort: O(n log n) average and worst case
```

**O(n²) — Quadratic:**
Every element interacts with every other element. Nested loops are the signal.
```python
def has_duplicate_brute(arr):
    for i in range(len(arr)):
        for j in range(i + 1, len(arr)):
            if arr[i] == arr[j]:
                return True
    return False
```

**O(2ⁿ) — Exponential:**
At each step you branch into 2 choices. Classic recursive Fibonacci without memoization.
```python
def fib(n):
    if n <= 1:
        return n
    return fib(n-1) + fib(n-2)  # Each call spawns 2 more calls
```

---

## Space Complexity

Space complexity = **extra memory** your algorithm uses (beyond the input itself).

| What you use | Space cost |
|-------------|-----------|
| A few variables | O(1) |
| A hash set/map of input | O(n) |
| A 2D matrix copy | O(n²) |
| Recursive call stack | O(depth) |

### Key Insight for Interviews
When the interviewer says "Can you do it in-place?", they want O(1) extra space. This usually means modifying the input array directly.

---

## How to Analyze Complexity — A Step-by-Step Method

### Step 1: Identify the dominant operation
What operation grows with input? Usually it's comparisons, iterations, or recursive calls.

### Step 2: Count the iterations
```
Single loop over n elements → O(n)
Nested loop (both over n) → O(n²)
Loop that halves work each time → O(log n)
Loop inside halving → O(n log n)
```

### Step 3: Drop constants and lower terms
- O(2n + 5) → O(n)
- O(n² + n) → O(n²)
- O(3n log n + 100n) → O(n log n)

### Step 4: Consider all paths for worst case
Big-O is about the **worst case** unless stated otherwise.

---

## Common Traps and Mistakes

### Trap 1: Hidden loops
```python
# This looks O(n) but string concatenation in Python creates new string each time
s = ""
for char in arr:
    s += char  # O(n) per concatenation → O(n²) total!

# Fix: use join
s = "".join(arr)  # O(n)
```

### Trap 2: Built-in method costs
```python
# list.pop(0) is O(n), not O(1)!
# Use collections.deque for O(1) popleft

# "x in list" is O(n)
# "x in set" is O(1)

# list.sort() is O(n log n), not free
```

### Trap 3: Recursive space
```python
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)
# Time: O(n), Space: O(n) due to call stack!
```

### Trap 4: Amortized complexity
- `list.append()` in Python is amortized O(1) — occasionally O(n) when resizing, but averages to O(1)
- Hash table insertions: amortized O(1)
- Important: Amortized ≠ Average. Amortized is a **guarantee** over a sequence.

---

## The Complexity Hierarchy — What Interviewers Want

For a typical interview problem (n ≈ 10⁴ to 10⁵):

| If constraint is... | Target complexity | Approach hint |
|--------------------|--------------------|---------------|
| n ≤ 10 | O(n!) or O(2ⁿ) | Brute force / backtracking OK |
| n ≤ 20 | O(2ⁿ) | Bitmask / subset enumeration |
| n ≤ 500 | O(n³) | Triple nested loop might work |
| n ≤ 5000 | O(n²) | DP with 2D table |
| n ≤ 10⁵ | O(n log n) | Sorting + scanning |
| n ≤ 10⁶ | O(n) | Hash map / two pointer / sliding window |
| n ≤ 10⁸ | O(log n) or O(1) | Math / binary search |

**This table is GOLD for interviews.** Look at the input constraint, and it tells you what complexity you need to hit.

---

## Real Interview Questions on Complexity

**Q1:** "Your solution uses O(n) extra space. Can you optimize to O(1)?"
**A:** This typically means: sort the array in-place, or use two pointers, or use bit manipulation.

**Q2:** "What's the complexity of your hash map solution?"
**A:** "O(n) time for building the map, O(1) average per lookup, O(n) space for the map. Worst case lookup is O(n) due to hash collisions, but with a good hash function this is practically O(1)."

**Q3:** "Is your recursive solution the most efficient?"
**A:** Check for overlapping subproblems → add memoization → drops from O(2ⁿ) to O(n) or O(n²).

---

## Practice Exercise

Analyze the complexity of this code:
```python
def mystery(arr):
    n = len(arr)
    result = set()
    arr.sort()  # What's this cost?
    for i in range(n):
        for j in range(i + 1, n):
            if arr[i] + arr[j] in result:  # What's this cost?
                return True
            result.add(arr[i] + arr[j])
    return False
```

**Answer:** 
- `arr.sort()` → O(n log n)
- Nested loops → O(n²) iterations
- `in result` (set lookup) → O(1) average
- **Total Time:** O(n²) (dominates the sort)
- **Space:** O(n²) worst case (set could store n² sums)

---

## Key Takeaways

1. Always state complexity after presenting your solution — don't wait to be asked
2. Use the constraint table to identify your target complexity BEFORE coding
3. Look for hidden costs in built-in methods
4. Space complexity includes the call stack for recursion
5. When optimizing: Can you trade space for time? (Hash map) Can you avoid redundant work? (DP/memoization)
