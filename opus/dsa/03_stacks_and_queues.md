# Stacks & Queues — FAANG Questions

## Core Concepts

### Stack (LIFO)
Use when you need to track **"most recent" state** — parentheses matching, undo operations, monotonic patterns.

### Queue (FIFO)
Use for **level-order processing**, BFS, and maintaining order.

### Monotonic Stack/Queue
The secret weapon for "next greater/smaller element" problems. Maintains elements in sorted order.

```
Stack Use Cases:                  Queue Use Cases:
─────────────────                 ─────────────────
Matching brackets                 BFS traversal
Expression evaluation             Level-order processing
Undo/History                      Sliding window problems
DFS (iterative)                   Task scheduling
Monotonic problems                Rate limiting
```

---

## Questions

### 🟢 Easy

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 1 | Valid Parentheses | A, M, G, MS | Stack: push open, pop on close |
| 2 | Min Stack | A, M, G | Two stacks or stack of pairs |
| 3 | Implement Queue using Stacks | A, M, G | Two stacks (lazy transfer) |
| 4 | Implement Stack using Queues | A, M | Push-costly or pop-costly |
| 5 | Baseball Game | A, G | Stack simulation |
| 6 | Next Greater Element I | A, G, M | Monotonic stack + hash map |
| 7 | Backspace String Compare | G, A, M | Stack or two pointers from end |
| 8 | Remove All Adjacent Duplicates in String | A, M | Stack |

---

### 🟡 Medium

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 9 | Daily Temperatures | A, G, M | Monotonic decreasing stack |
| 10 | Evaluate Reverse Polish Notation | A, M, G | Stack: push nums, compute on operator |
| 11 | Generate Parentheses | A, M, G | Backtracking (can model with stack) |
| 12 | Decode String | A, G, M | Stack of (string, count) pairs |
| 13 | Asteroid Collision | A, G | Stack simulation with collision rules |
| 14 | Remove K Digits | A, G, M | Monotonic stack (remove larger digits) |
| 15 | 132 Pattern | A, G | Monotonic stack from right |
| 16 | Next Greater Element II (circular) | A, G, M | Monotonic stack, iterate 2x |
| 17 | Simplify Path | A, M, G | Stack: split by '/', handle '..' |
| 18 | Online Stock Span | A, G | Monotonic stack |
| 19 | Car Fleet | G, A | Sort by position, stack by time |
| 20 | Flatten Nested List Iterator | M, A, G | Stack of iterators |
| 21 | Exclusive Time of Functions | M, A | Stack tracking start times |
| 22 | Basic Calculator II | A, M, G | Stack with operator precedence |
| 23 | Validate Stack Sequences | A, G | Simulate push/pop |
| 24 | Sum of Subarray Minimums | A, G | Monotonic stack + contribution |
| 25 | Circular Queue Design | A, M | Array with head/tail pointers |

---

### 🔴 Hard

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 26 | Largest Rectangle in Histogram | G, A, M | Monotonic stack (find boundaries) |
| 27 | Maximal Rectangle | G, A, M | Reduce to histogram per row |
| 28 | Trapping Rain Water | G, A, M | Monotonic stack approach |
| 29 | Basic Calculator (with parentheses) | A, M, G | Stack of (result, sign) |
| 30 | Longest Valid Parentheses | A, G, M | Stack storing indices |
| 31 | Maximum Frequency Stack | G, A, M | Stack per frequency level |
| 32 | Shortest Subarray with Sum at Least K | G, A | Monotonic deque + prefix sum |

---

## Monotonic Stack Template

```python
def next_greater_element(nums):
    """Find next greater element for each position"""
    n = len(nums)
    result = [-1] * n
    stack = []  # stores indices
    
    for i in range(n):
        # Pop elements smaller than current
        while stack and nums[stack[-1]] < nums[i]:
            idx = stack.pop()
            result[idx] = nums[i]
        stack.append(i)
    
    return result
```

## Stack for Expressions Template

```python
def calculate(s):
    """Basic Calculator pattern"""
    stack = []
    num = 0
    sign = 1
    result = 0
    
    for char in s:
        if char.isdigit():
            num = num * 10 + int(char)
        elif char in '+-':
            result += sign * num
            num = 0
            sign = 1 if char == '+' else -1
        elif char == '(':
            stack.append(result)
            stack.append(sign)
            result = 0
            sign = 1
        elif char == ')':
            result += sign * num
            num = 0
            result *= stack.pop()  # sign
            result += stack.pop()  # previous result
    
    return result + sign * num
```
