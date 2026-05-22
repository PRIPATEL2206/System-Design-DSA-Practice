# DSA Final Revision — Read 20 Min Before Interview

## Pattern Recognition (Identify in < 30 seconds)

```
"Find pair/sum/complement"           → HashMap or Two Pointer
"Subarray sum equals K"              → Prefix Sum + HashMap
"Longest/shortest subarray"          → Sliding Window
"Find in sorted array"               → Binary Search
"Find in rotated array"              → Modified Binary Search
"Top K / Kth largest"                → Heap (min-heap of K)
"Shortest path (unweighted)"         → BFS
"Connected components"               → DFS or Union-Find
"Detect cycle (directed)"            → DFS with coloring
"Detect cycle (undirected)"          → Union-Find
"Topological ordering"               → Kahn's (BFS) or DFS
"All permutations/combinations"      → Backtracking
"Count ways / min cost"              → Dynamic Programming
"Merge intervals"                    → Sort by start + merge
"Next greater/smaller"               → Monotonic Stack
"Level order / layer by layer"       → BFS with queue
"Validate BST"                       → Inorder or range check
"LCA (Lowest Common Ancestor)"       → Recursive divide
"Palindrome"                         → Two pointer or expand
"Anagram"                            → Frequency array
"Linked list cycle"                  → Floyd's slow/fast
"Find middle of LL"                  → Slow/fast pointer
"Serialize/clone structure"          → BFS or Preorder DFS
"Design with O(1) operations"       → HashMap + LinkedList
```

---

## Complexity Cheat Sheet

```
O(1)        HashMap lookup, array index, stack push/pop
O(log n)    Binary search, balanced BST ops, heap ops
O(n)        Linear scan, two pointer, sliding window
O(n log n)  Sorting (merge/quick), heap sort
O(n²)       Nested loops, brute force pairs
O(2ⁿ)       Subsets, recursive without memo
O(n!)       Permutations

Space:
O(1)        In-place (two pointer, etc.)
O(n)        HashMap, queue, recursion stack
O(n²)       2D DP table
```

---

## Templates (Copy-Paste Into Your Brain)

### Sliding Window
```python
def sliding_window(s):
    left = 0
    window = {}
    result = 0
    for right in range(len(s)):
        window[s[right]] = window.get(s[right], 0) + 1
        while INVALID(window):
            window[s[left]] -= 1
            if window[s[left]] == 0: del window[s[left]]
            left += 1
        result = max(result, right - left + 1)
    return result
```

### Binary Search
```python
def binary_search(lo, hi):
    while lo < hi:
        mid = (lo + hi) // 2
        if condition(mid):
            hi = mid
        else:
            lo = mid + 1
    return lo
```

### BFS
```python
from collections import deque
def bfs(start):
    queue = deque([start])
    visited = {start}
    while queue:
        node = queue.popleft()
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
```

### DFS (Backtracking)
```python
def backtrack(path, choices):
    if IS_SOLUTION(path):
        result.append(path[:])
        return
    for choice in choices:
        if VALID(choice):
            path.append(choice)
            backtrack(path, NEXT_CHOICES)
            path.pop()
```

### Topo Sort (Kahn's)
```python
def topo_sort(graph, n):
    indegree = [0] * n
    for u in graph:
        for v in graph[u]:
            indegree[v] += 1
    queue = deque([i for i in range(n) if indegree[i] == 0])
    order = []
    while queue:
        node = queue.popleft()
        order.append(node)
        for nei in graph[node]:
            indegree[nei] -= 1
            if indegree[nei] == 0:
                queue.append(nei)
    return order if len(order) == n else []
```

### DP (1D)
```python
dp[i] = best answer considering first i elements
dp[0] = base case
for i in range(1, n):
    dp[i] = max/min(dp[i-1] + ..., dp[i-2] + ..., ...)
return dp[n-1]
```

### Union-Find
```python
parent = list(range(n))
def find(x):
    if parent[x] != x:
        parent[x] = find(parent[x])
    return parent[x]
def union(x, y):
    parent[find(x)] = find(y)
```

### Monotonic Stack
```python
stack = []
for i in range(n):
    while stack and nums[stack[-1]] < nums[i]:
        idx = stack.pop()
        result[idx] = nums[i]  # next greater
    stack.append(i)
```

---

## Edge Cases (Always Check!)

```
□ Empty input ([], "", None)
□ Single element
□ Two elements
□ All same elements
□ Already sorted (asc and desc)
□ Negative numbers
□ Integer overflow (use Python → no issue, but mention it)
□ Duplicates
□ Very large input (will O(n²) TLE?)
```

---

## Communication Scripts

```
Start: "Let me make sure I understand the problem correctly..."
Stuck: "Let me think about what patterns might apply here..."
Trade-off: "We could do X in O(n²) or use extra space for O(n)..."
Done: "Let me trace through an example to verify..."
Bug: "Ah, I see the issue — [fix]. Good catch from the test case."
```
