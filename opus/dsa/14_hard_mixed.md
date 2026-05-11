# Hard Mixed Practice — FAANG Questions

## Purpose
These are the hardest problems frequently asked at FAANG. Practice these after completing the topic-wise files. Each problem requires combining multiple techniques.

---

## Questions

### Top 50 Hard Problems (Sorted by Frequency at FAANG)

| # | Problem | Companies | Topics | Key Idea |
|---|---------|-----------|--------|----------|
| 1 | Median of Two Sorted Arrays | G, A, M, MS | Binary Search | BS on partition, O(log(min(m,n))) |
| 2 | Merge K Sorted Lists | A, M, G | Heap, Linked List | Min-heap of K heads |
| 3 | Trapping Rain Water | G, A, M, AP | Two Pointer, Stack | Two pointers with left_max, right_max |
| 4 | Regular Expression Matching | G, A, M | DP, String | 2D DP with '.' and '*' rules |
| 5 | Word Ladder II | A, G, M | BFS, DFS, Graph | BFS distance + DFS paths |
| 6 | Minimum Window Substring | M, A, G | Sliding Window | Expand/shrink with freq tracking |
| 7 | Largest Rectangle in Histogram | G, A, M | Stack | Monotonic stack boundaries |
| 8 | Binary Tree Maximum Path Sum | A, M, G | Tree, DFS | Global max, return single path |
| 9 | Word Break II | A, M, G | DP, Backtracking | Memo + backtrack all sentences |
| 10 | LRU Cache | A, M, G, MS, AP | Design, Hash + LL | HashMap + Doubly Linked List |
| 11 | Serialize/Deserialize Binary Tree | A, M, G | Tree | Preorder with null markers |
| 12 | Alien Dictionary | A, G, M | Graph, Topo Sort | Build graph from word order |
| 13 | Basic Calculator | A, M, G | Stack | Recursion or stack with signs |
| 14 | Sliding Window Maximum | A, G, M | Deque | Monotonic decreasing deque |
| 15 | N-Queens | A, G, M | Backtracking | Row-by-row with col/diag checks |
| 16 | Course Schedule II | A, M, G | Graph, Topo Sort | Kahn's algorithm |
| 17 | Find Median from Data Stream | A, M, G, MS | Heap | Two heaps (max-left, min-right) |
| 18 | Longest Increasing Subsequence (O(n log n)) | A, M, G | Binary Search, DP | Patience sorting |
| 19 | Edit Distance | A, M, G | DP, String | 2D DP: insert/delete/replace |
| 20 | Critical Connections in Network | A, G | Graph, DFS | Tarjan's bridge algorithm |
| 21 | Burst Balloons | G, A | Interval DP | Last balloon to burst |
| 22 | Palindrome Pairs | A, G | Trie, String | Trie of reversed words |
| 23 | Word Search II | A, G, M | Trie, Backtracking | Build trie, DFS on grid |
| 24 | Swim in Rising Water | G, A | Binary Search, BFS | BS on answer + BFS validation |
| 25 | Minimum Cost to Hire K Workers | G | Heap, Greedy | Sort by ratio, max-heap for cost |
| 26 | Employee Free Time | A, G, M | Heap, Intervals | Merge K sorted + find gaps |
| 27 | Reverse Nodes in K-Group | A, M, G | Linked List | Reverse K nodes iteratively |
| 28 | Longest Valid Parentheses | A, G, M | Stack, DP | Stack with indices |
| 29 | First Missing Positive | A, G, M | Array | Cyclic sort to correct positions |
| 30 | Shortest Path Visiting All Nodes | G | BFS, Bitmask | BFS with state = (node, visited_mask) |
| 31 | Count of Smaller Numbers After Self | G, A | Merge Sort, BIT | Modified merge sort |
| 32 | Russian Doll Envelopes | G, A | Sort, Binary Search | Sort + LIS on heights |
| 33 | Design In-Memory File System | G, A | Trie, Design | Trie where nodes = directories |
| 34 | Maximum Frequency Stack | G, A, M | Stack, Hash | Frequency-indexed stacks |
| 35 | Smallest Range Covering K Lists | G, A | Heap | Min-heap + track global max |
| 36 | Number of Islands II (Online) | G, A | Union-Find | Dynamic UF with additions |
| 37 | Text Justification | G, A | String, Greedy | Greedy fill + padding |
| 38 | Wildcard Matching | G, A, M | DP | 2D DP with '?' and '*' |
| 39 | Reconstruct Itinerary | G, A, M | Graph | Hierholzer's Eulerian path |
| 40 | LFU Cache | A, G, M | Design | HashMap + freq doubly-LL |
| 41 | Maximal Rectangle | G, A, M | Stack, DP | Row histogram + largest rectangle |
| 42 | Concatenated Words | A, G | Trie, DP | Word break on each word |
| 43 | Minimum Number of Refueling Stops | G, A | Heap, Greedy | Max-heap of passed stations |
| 44 | Making A Large Island | G, A, M | Union-Find, DFS | Label islands + try flip each 0 |
| 45 | Subarrays with K Different Integers | G, A, M | Sliding Window | atMost(K) - atMost(K-1) |
| 46 | Longest Substring with At Most K Distinct | G, A, M | Sliding Window | Window + hash map size ≤ K |
| 47 | Minimum Window Subsequence | G, M | Two Pointer, DP | Forward/backward scan |
| 48 | Expression Add Operators | G, A, M | Backtracking | Insert +, -, * between digits |
| 49 | Count of Range Sum | G, A | Merge Sort | Count during merge |
| 50 | Shortest Subarray with Sum at Least K | G, A | Deque, Prefix Sum | Monotonic deque on prefix |

---

## Daily Practice Plan for Hard Problems

**Week 1:** Problems 1-10 (2 per day on weekdays)
**Week 2:** Problems 11-20
**Week 3:** Problems 21-35 (3 per day — these build on patterns you've mastered)
**Week 4:** Problems 36-50 + retry all ❌ marked problems

---

## Tips for Hard Problems

1. **Don't panic** — Hard problems combine 2-3 medium concepts
2. **Identify the combination** — "This is BFS + bitmask" or "Sort + binary search"
3. **Time yourself** — 30-40 min max. If stuck after 20 min, read the approach (not code)
4. **Implement from approach** — Don't memorize solutions, understand the WHY
5. **Pattern match** — Most hards are variations of patterns from medium problems
