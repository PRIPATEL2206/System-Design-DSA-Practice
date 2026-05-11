# Pattern Recognition Guide — The Secret to Solving Any Problem

## The Core Truth About Coding Interviews

There are only ~15 fundamental patterns. Every LeetCode problem is a variation of one (or a combination of two). Learn to recognize the pattern, and the solution becomes obvious.

---

## Pattern Recognition Flowchart

```
START: Read the problem
│
├── Input is SORTED array?
│   ├── Find pair/triplet → Two Pointers
│   ├── Find element → Binary Search
│   └── Merge sorted inputs → Merge technique
│
├── Input is UNSORTED array?
│   ├── Need O(1) lookup → Hash Map/Set
│   ├── Contiguous subarray property → Sliding Window
│   ├── Subarray sum → Prefix Sum + Hash Map
│   └── Global max/min → Kadane's or DP
│
├── Input is STRING?
│   ├── Anagram/frequency → Counter/Hash Map
│   ├── Substring with window property → Sliding Window
│   ├── Palindrome → Two Pointers / Expand from center
│   └── Pattern matching → DP
│
├── Input is LINKED LIST?
│   ├── Cycle detection → Fast/Slow pointers
│   ├── Find middle/kth from end → Fast/Slow or two pass
│   └── Reverse/merge → Pointer manipulation
│
├── Input is TREE?
│   ├── Traversal needed → DFS (in/pre/post) or BFS
│   ├── Path problem → DFS with global variable
│   ├── Level-by-level → BFS with queue
│   └── BST search → Exploit sorted property
│
├── Input is GRAPH?
│   ├── Shortest path (unweighted) → BFS
│   ├── Shortest path (weighted) → Dijkstra
│   ├── Connectivity → Union-Find or DFS
│   ├── Ordering with dependencies → Topological Sort
│   └── All paths / cycle detection → DFS
│
├── Input is MATRIX/GRID?
│   ├── Connected regions → DFS/BFS flood fill
│   ├── Shortest path in grid → BFS
│   └── Path counting → Grid DP
│
├── Need TOP K elements?
│   └── Heap (priority queue)
│
├── Need ALL combinations/permutations?
│   └── Backtracking
│
├── "Minimum/Maximum" with choices at each step?
│   ├── Overlapping subproblems → DP
│   └── Local choice is globally optimal → Greedy
│
└── "Design a data structure"?
    └── Combine: Hash Map + Linked List/Heap/Array
```

---

## The 15 Master Patterns — Quick Reference

### 1. Sliding Window
**Signal:** Contiguous subarray/substring, "longest/shortest with condition"
**Template:**
```python
left = 0
for right in range(len(arr)):
    # Add arr[right] to window
    while window_invalid():
        # Remove arr[left] from window
        left += 1
    # Update answer
```
**Problems:** Longest substring without repeat, minimum window substring, max sum subarray of size k

### 2. Two Pointers
**Signal:** Sorted input, find pairs, remove duplicates, partition
**Template:**
```python
left, right = 0, len(arr) - 1
while left < right:
    # Compare, move left or right
```
**Problems:** Two sum (sorted), three sum, container with most water, remove duplicates

### 3. Fast & Slow Pointers
**Signal:** Linked list cycle, find middle, happy number
**Template:**
```python
slow = fast = head
while fast and fast.next:
    slow = slow.next
    fast = fast.next.next
```

### 4. Binary Search
**Signal:** Sorted array, "find minimum/maximum that satisfies", search space reduction
**Template:**
```python
lo, hi = min_possible, max_possible
while lo < hi:
    mid = (lo + hi) // 2
    if condition(mid):
        hi = mid
    else:
        lo = mid + 1
return lo
```
**Problems:** Search rotated array, find peak, koko eating bananas, split array largest sum

### 5. BFS (Breadth-First Search)
**Signal:** Shortest path, level-order, minimum steps, nearest
**Template:**
```python
queue = deque([start])
visited = {start}
level = 0
while queue:
    for _ in range(len(queue)):
        node = queue.popleft()
        for neighbor in get_neighbors(node):
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    level += 1
```

### 6. DFS (Depth-First Search)
**Signal:** Explore all paths, connected components, tree traversal, backtracking
**Template:**
```python
def dfs(node, visited):
    if base_case:
        return
    visited.add(node)
    for neighbor in graph[node]:
        if neighbor not in visited:
            dfs(neighbor, visited)
```

### 7. Topological Sort
**Signal:** Dependencies, ordering, course schedule, build order
**Key:** Kahn's algorithm (BFS with in-degree) or DFS with finish time

### 8. Hash Map/Set
**Signal:** "Find if exists", "count occurrences", "group by", O(1) lookup needed
**Problems:** Two sum, group anagrams, subarray sum equals k

### 9. Monotonic Stack/Queue
**Signal:** "Next greater/smaller element", "max in sliding window"
**Key:** Maintain increasing or decreasing order in stack

### 10. Heap / Priority Queue
**Signal:** "Top K", "Kth largest/smallest", "merge sorted", "schedule with priorities"
**Key:** Min-heap of size k for kth largest, max-heap for kth smallest

### 11. Dynamic Programming
**Signal:** "Count ways", "minimum/maximum", optimal substructure + overlapping subproblems
**Key:** Define state → find recurrence → base case → fill table

### 12. Greedy
**Signal:** Intervals, scheduling, "local choice = global optimal", sorting + one pass
**Key:** Sort by some criterion, make greedy choice at each step

### 13. Backtracking
**Signal:** "All combinations/permutations/subsets", "generate all", constraint satisfaction
**Key:** Choose → explore → unchoose

### 14. Union-Find
**Signal:** "Connected components", "group membership", "redundant connection"
**Key:** find() with path compression, union() with rank

### 15. Trie (Prefix Tree)
**Signal:** "Prefix matching", "autocomplete", "word search in dictionary"
```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()
    
    def insert(self, word):
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        node.is_end = True
    
    def search(self, word):
        node = self.root
        for char in word:
            if char not in node.children:
                return False
            node = node.children[char]
        return node.is_end
    
    def starts_with(self, prefix):
        node = self.root
        for char in prefix:
            if char not in node.children:
                return False
            node = node.children[char]
        return True
```

---

## Pattern Combinations (Hard Problems)

Hard problems often combine 2 patterns:

| Problem | Pattern Combination |
|---------|-------------------|
| Trapping Rain Water | Two Pointers + Monotonic Stack |
| Word Ladder | BFS + Hash Set |
| Alien Dictionary | Topological Sort + Hash Map |
| Serialize/Deserialize BST | DFS + Queue |
| LRU Cache | Hash Map + Doubly Linked List |
| Sliding Window Maximum | Sliding Window + Monotonic Deque |
| Word Search II | Trie + Backtracking |
| Merge K Sorted Lists | Heap + Linked List |
| Course Schedule II | Topological Sort + BFS |

---

## Practice Strategy: 75 Problems That Cover All Patterns

Solve in this order (builds on previous patterns):

**Week 1-2: Arrays & Hashing (foundation)**
1. Two Sum, 2. Valid Anagram, 3. Group Anagrams, 4. Top K Frequent, 5. Product of Array Except Self, 6. Longest Consecutive Sequence

**Week 2-3: Two Pointers & Sliding Window**
7. Valid Palindrome, 8. Three Sum, 9. Container With Most Water, 10. Best Time Buy/Sell Stock, 11. Longest Substring Without Repeat, 12. Minimum Window Substring

**Week 3-4: Stack & Binary Search**
13. Valid Parentheses, 14. Min Stack, 15. Daily Temperatures, 16. Binary Search, 17. Search Rotated Array, 18. Find Minimum Rotated

**Week 4-5: Linked List & Trees**
19. Reverse Linked List, 20. Merge Two Lists, 21. LRU Cache, 22. Invert Tree, 23. Max Depth, 24. Same Tree, 25. Subtree of Another, 26. LCA, 27. Level Order, 28. Validate BST

**Week 5-6: Graphs**
29. Number of Islands, 30. Clone Graph, 31. Pacific Atlantic, 32. Course Schedule, 33. Graph Valid Tree, 34. Word Ladder

**Week 6-7: Dynamic Programming**
35. Climbing Stairs, 36. House Robber, 37. Coin Change, 38. Longest Increasing Subsequence, 39. LCS, 40. Word Break, 41. Decode Ways, 42. Unique Paths, 43. Edit Distance

**Week 7-8: Backtracking & Greedy**
44. Subsets, 45. Combination Sum, 46. Permutations, 47. Word Search, 48. N-Queens, 49. Merge Intervals, 50. Non-Overlapping Intervals, 51. Jump Game

**Week 8-9: Heap & Advanced**
52. Kth Largest, 53. Merge K Lists, 54. Find Median Stream, 55. Task Scheduler

**Week 9-10: Mixed Hard Problems**
56-75: Trapping Rain Water, Median of Two Sorted Arrays, Serialize Tree, Word Search II, Alien Dictionary, etc.

---

## The "I'm Stuck" Algorithm

When you can't identify the pattern:

1. **Look at constraints** — n ≤ 10⁵ means O(n log n) or better
2. **Look at the ask** — "All" = backtracking, "Minimum/Maximum" = DP or greedy, "Exists" = hash
3. **Look at the structure** — Sorted? Tree? Graph? Grid?
4. **Try brute force** — What data structure would speed up the brute force?
5. **Simplify** — Solve for n=1, n=2, n=3. See the pattern.

---

## Key Takeaway

Don't solve 500 random problems. Solve 75 problems deliberately, one pattern at a time. When you see a new problem, your brain should instantly think: "This is a sliding window problem" or "This needs topological sort." That pattern recognition is what separates candidates who get offers from those who don't.
