# DSA Quick Revision Cheatsheet — Last Day Before Interview

## Time Complexity Quick Reference
```
O(1)       → Hash lookup, array access
O(log n)   → Binary search, balanced BST ops
O(n)       → Linear scan, hash map build
O(n log n) → Sorting (merge/heap/tim sort)
O(n²)      → Nested loops, brute force pairs
O(2ⁿ)      → Subsets, recursive without memo
O(n!)      → Permutations
```

## Pattern → Trigger → Template

### 1. Two Pointers
**Trigger:** Sorted array, find pair, remove duplicates
```python
left, right = 0, len(arr) - 1
while left < right:
    if condition: left += 1
    else: right -= 1
```

### 2. Sliding Window
**Trigger:** Subarray/substring with constraint, "longest/shortest"
```python
left = 0
for right in range(n):
    # expand window
    while invalid:
        # shrink from left
        left += 1
    ans = max(ans, right - left + 1)
```

### 3. Binary Search
**Trigger:** Sorted, "minimum that satisfies", search space
```python
lo, hi = 0, n - 1
while lo < hi:
    mid = (lo + hi) // 2
    if condition(mid): hi = mid
    else: lo = mid + 1
return lo
```

### 4. BFS
**Trigger:** Shortest path, level-order, minimum steps
```python
queue = deque([start])
visited = {start}
while queue:
    for _ in range(len(queue)):
        node = queue.popleft()
        for next in neighbors(node):
            if next not in visited:
                visited.add(next)
                queue.append(next)
```

### 5. DFS
**Trigger:** All paths, connected components, tree traversal
```python
def dfs(node):
    if not node: return
    # process node
    dfs(node.left)
    dfs(node.right)
```

### 6. Monotonic Stack
**Trigger:** Next greater/smaller element
```python
stack = []
for i in range(n):
    while stack and arr[i] > arr[stack[-1]]:
        idx = stack.pop()
        result[idx] = arr[i]
    stack.append(i)
```

### 7. Topological Sort (Kahn's BFS)
**Trigger:** Dependencies, ordering, course schedule
```python
queue = deque([n for n in range(N) if indegree[n] == 0])
order = []
while queue:
    node = queue.popleft()
    order.append(node)
    for next in graph[node]:
        indegree[next] -= 1
        if indegree[next] == 0:
            queue.append(next)
```

### 8. Union-Find
**Trigger:** Connected components, group membership
```python
def find(x):
    if parent[x] != x:
        parent[x] = find(parent[x])
    return parent[x]

def union(x, y):
    px, py = find(x), find(y)
    if px != py: parent[px] = py
```

### 9. Backtracking
**Trigger:** All combinations/permutations, constraint satisfaction
```python
def backtrack(start, path):
    if valid(path):
        result.append(path[:])
        return
    for i in range(start, n):
        path.append(arr[i])
        backtrack(i + 1, path)
        path.pop()
```

### 10. Dynamic Programming
**Trigger:** Optimal, count ways, overlapping subproblems
```python
# 1D: dp[i] = answer for first i elements
# 2D: dp[i][j] = answer for subproblem (i, j)
# Knapsack: dp[w] = max(dp[w], dp[w-wt] + val)
# LCS: dp[i][j] = dp[i-1][j-1]+1 if match else max(dp[i-1][j], dp[i][j-1])
```

---

## Data Structure Operations

| Structure | Access | Search | Insert | Delete | Notes |
|-----------|--------|--------|--------|--------|-------|
| Array | O(1) | O(n) | O(n) | O(n) | O(1) at end |
| Hash Map | — | O(1)* | O(1)* | O(1)* | *Average case |
| BST | — | O(log n)* | O(log n)* | O(log n)* | *Balanced |
| Heap | O(1) top | O(n) | O(log n) | O(log n) | Min/max only |
| Stack | O(1) top | O(n) | O(1) | O(1) | LIFO |
| Queue | O(1) front | O(n) | O(1) | O(1) | FIFO |
| Linked List | O(n) | O(n) | O(1)* | O(1)* | *With pointer |

---

## Must-Remember Tricks

```python
# Swap without temp
a, b = b, a

# Check if power of 2
n & (n - 1) == 0

# Get lowest set bit
n & (-n)

# Integer overflow in other languages
mid = lo + (hi - lo) // 2  # Instead of (lo + hi) // 2

# Infinity
float('inf'), float('-inf')

# Default dict
from collections import defaultdict, Counter, deque

# Sort by custom key
intervals.sort(key=lambda x: x[1])

# Reverse iterate
for i in range(n-1, -1, -1):

# Enumerate with index
for i, val in enumerate(arr):

# Zip two lists
for a, b in zip(list1, list2):

# List comprehension with condition
[x for x in arr if x > 0]

# Flatten 2D
[item for row in matrix for item in row]
```

---

## Top 20 Problems to Review Night Before

| # | Problem | Pattern | Time |
|---|---------|---------|------|
| 1 | Two Sum | Hash Map | O(n) |
| 2 | Valid Parentheses | Stack | O(n) |
| 3 | Merge Intervals | Sort + sweep | O(n log n) |
| 4 | Best Time Buy/Sell Stock | Kadane | O(n) |
| 5 | Longest Substring No Repeat | Sliding Window | O(n) |
| 6 | 3Sum | Sort + Two Pointers | O(n²) |
| 7 | Reverse Linked List | Pointer swap | O(n) |
| 8 | LRU Cache | HashMap + DLL | O(1) |
| 9 | Number of Islands | Grid DFS | O(mn) |
| 10 | Course Schedule | Topological Sort | O(V+E) |
| 11 | Binary Tree Level Order | BFS | O(n) |
| 12 | Validate BST | DFS with bounds | O(n) |
| 13 | Coin Change | DP | O(n×amount) |
| 14 | Longest Increasing Subseq | DP/Binary Search | O(n log n) |
| 15 | Word Break | DP + Hash Set | O(n²) |
| 16 | Top K Frequent | Heap/Bucket | O(n log k) |
| 17 | Clone Graph | BFS + Hash Map | O(V+E) |
| 18 | Merge K Sorted Lists | Min Heap | O(N log k) |
| 19 | Lowest Common Ancestor | DFS | O(n) |
| 20 | Maximum Path Sum (Tree) | DFS + global max | O(n) |

---

## Edge Cases Checklist
Before submitting ANY solution, check:
- [ ] Empty input (`[]`, `""`, `None`)
- [ ] Single element
- [ ] All same elements
- [ ] Negative numbers
- [ ] Already sorted / reverse sorted
- [ ] Duplicates
- [ ] Very large input (does your solution TLE?)
- [ ] Integer overflow (for non-Python languages)
