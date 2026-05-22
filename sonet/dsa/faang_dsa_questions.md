# FAANG DSA Question Bank — Prince Patel
> 300 questions · 30 days · 10 questions/day
> Companies: **G**=Google · **A**=Amazon · **M**=Meta · **Ap**=Apple · **N**=Netflix · **Mi**=Microsoft

## Status Legend
| Mark | Meaning |
|------|---------|
| `[ ]` | Not started |
| `[~]` | Attempted — need to revisit |
| `[x]` | Solved cleanly |
| `[!]` | Important — must revisit before interview |

## Topic Roadmap
| Days | Topic |
|------|-------|
| 01–03 | Arrays & Hashing |
| 04–05 | Two Pointers & Sliding Window |
| 06–08 | Linked Lists |
| 09–10 | Stacks & Queues |
| 11 | Binary Search |
| 12–15 | Trees & BST |
| 16–17 | Heaps & Priority Queues |
| 18–20 | Graphs |
| 21–25 | Dynamic Programming |
| 26–27 | Backtracking |
| 28 | Tries |
| 29 | Bit Manipulation |
| 30 | Math & Mixed Review |

---

## DAY 01 — Arrays: Basics
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Two Sum | 1 | 🟢 Easy | G, A, M | Hash Map |
| `[ ]` | Best Time to Buy and Sell Stock | 121 | 🟢 Easy | A, G, M | Min tracking |
| `[ ]` | Contains Duplicate | 217 | 🟢 Easy | A, Ap, N | Hash Set |
| `[ ]` | Product of Array Except Self | 238 | 🟡 Medium | A, G, M | Prefix/Suffix product |
| `[ ]` | Maximum Subarray | 53 | 🟡 Medium | A, G, M | Kadane's Algorithm |
| `[ ]` | Maximum Product Subarray | 152 | 🟡 Medium | A, G | Track min & max |
| `[ ]` | Find Minimum in Rotated Sorted Array | 153 | 🟡 Medium | A, G, Ap | Binary Search |
| `[ ]` | Search in Rotated Sorted Array | 33 | 🟡 Medium | A, G, Ap | Binary Search |
| `[ ]` | 3Sum | 15 | 🟡 Medium | G, A, M | Sort + Two Pointers |
| `[ ]` | Container With Most Water | 11 | 🟡 Medium | G, A | Two Pointers |

---

## DAY 02 — Arrays: Intermediate
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Trapping Rain Water | 42 | 🔴 Hard | G, A, M | Two Pointers / Stack |
| `[ ]` | Jump Game | 55 | 🟡 Medium | A, G | Greedy |
| `[ ]` | Jump Game II | 45 | 🟡 Medium | A, G | Greedy (min jumps) |
| `[ ]` | Merge Intervals | 56 | 🟡 Medium | G, A, M | Sort + Merge |
| `[ ]` | Insert Interval | 57 | 🟡 Medium | G, M | Greedy / Linear scan |
| `[ ]` | Non-overlapping Intervals | 435 | 🟡 Medium | G | Greedy (earliest end) |
| `[ ]` | Meeting Rooms II | 253 | 🟡 Medium | G, A, M | Min-Heap |
| `[ ]` | Rotate Array | 189 | 🟡 Medium | A, G, M | 3-step reversal |
| `[ ]` | Find the Duplicate Number | 287 | 🟡 Medium | A, G, Ap | Floyd's Cycle |
| `[ ]` | Spiral Matrix | 54 | 🟡 Medium | A, G, Ap | Simulation with boundaries |

---

## DAY 03 — Arrays: Advanced + Hashing
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Longest Consecutive Sequence | 128 | 🟡 Medium | G, A, M | Hash Set + O(n) |
| `[ ]` | Set Matrix Zeroes | 73 | 🟡 Medium | A, G, M | In-place flags |
| `[ ]` | Group Anagrams | 49 | 🟡 Medium | A, G, M | Sorted-key Hash Map |
| `[ ]` | Valid Anagram | 242 | 🟢 Easy | A, G, M | Char frequency |
| `[ ]` | Top K Frequent Elements | 347 | 🟡 Medium | G, A, M | Bucket Sort / Heap |
| `[ ]` | First Missing Positive | 41 | 🔴 Hard | A, G | Index-as-hash |
| `[ ]` | Subarray Sum Equals K | 560 | 🟡 Medium | G, A, M | Prefix Sum + Hash |
| `[ ]` | Sort Colors | 75 | 🟡 Medium | A, G | Dutch National Flag |
| `[ ]` | Majority Element | 169 | 🟢 Easy | A, G, M | Boyer-Moore Voting |
| `[ ]` | Encode and Decode Strings | 271 | 🟡 Medium | G, A | Length-prefix encoding |

---

## DAY 04 — Two Pointers
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Valid Palindrome | 125 | 🟢 Easy | M, A, G | Two Pointers |
| `[ ]` | Two Sum II — Input Array Is Sorted | 167 | 🟡 Medium | A, G | Two Pointers |
| `[ ]` | 3Sum Closest | 16 | 🟡 Medium | A, G | Sort + Two Pointers |
| `[ ]` | 4Sum | 18 | 🟡 Medium | A, G | Sort + Two Pointers |
| `[ ]` | Remove Duplicates from Sorted Array | 26 | 🟢 Easy | A, G, M | Slow/Fast pointer |
| `[ ]` | Move Zeroes | 283 | 🟢 Easy | M, A | Two Pointers |
| `[ ]` | Reverse String | 344 | 🟢 Easy | A, G, M | Two Pointers |
| `[ ]` | Is Subsequence | 392 | 🟢 Easy | A, G, M | Two Pointers |
| `[ ]` | Minimum Size Subarray Sum | 209 | 🟡 Medium | A, G | Sliding Window |
| `[ ]` | Squares of a Sorted Array | 977 | 🟢 Easy | G, A, M | Two Pointers from ends |

---

## DAY 05 — Sliding Window
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Longest Substring Without Repeating Characters | 3 | 🟡 Medium | A, G, M | Hash + Sliding Window |
| `[ ]` | Longest Repeating Character Replacement | 424 | 🟡 Medium | G, A | Window + max freq |
| `[ ]` | Permutation in String | 567 | 🟡 Medium | A, G, M | Fixed-size window |
| `[ ]` | Find All Anagrams in a String | 438 | 🟡 Medium | A, G, M | Fixed-size window |
| `[ ]` | Minimum Window Substring | 76 | 🔴 Hard | A, G, M | Variable window + freq |
| `[ ]` | Sliding Window Maximum | 239 | 🔴 Hard | A, G | Monotonic Deque |
| `[ ]` | Max Consecutive Ones III | 1004 | 🟡 Medium | A, G | Sliding Window (k zeros) |
| `[ ]` | Fruit Into Baskets | 904 | 🟡 Medium | A, G | Sliding Window (2 types) |
| `[ ]` | Longest Subarray of 1's After Deleting One Element | 1493 | 🟡 Medium | A, G | Sliding Window |
| `[ ]` | Number of Substrings Containing All Three Characters | 1358 | 🟡 Medium | G, A | Sliding Window |

---

## DAY 06 — Linked Lists: Basics
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Reverse Linked List | 206 | 🟢 Easy | A, G, M | Iterative / Recursive |
| `[ ]` | Merge Two Sorted Lists | 21 | 🟢 Easy | A, G, M | Two Pointers |
| `[ ]` | Linked List Cycle | 141 | 🟢 Easy | A, G, M | Floyd's Cycle Detection |
| `[ ]` | Linked List Cycle II | 142 | 🟡 Medium | A, G | Floyd's Cycle (entry point) |
| `[ ]` | Palindrome Linked List | 234 | 🟢 Easy | A, G, M | Find mid + Reverse half |
| `[ ]` | Remove Nth Node From End of List | 19 | 🟡 Medium | A, G, M | Two Pointers (gap n) |
| `[ ]` | Middle of the Linked List | 876 | 🟢 Easy | A, G | Fast/Slow Pointer |
| `[ ]` | Intersection of Two Linked Lists | 160 | 🟢 Easy | A, G, M | Two Pointers (align) |
| `[ ]` | Remove Duplicates from Sorted List | 83 | 🟢 Easy | A, G | One Pass |
| `[ ]` | Merge k Sorted Lists | 23 | 🔴 Hard | A, G, M | Min-Heap |

---

## DAY 07 — Linked Lists: Intermediate
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Add Two Numbers | 2 | 🟡 Medium | A, G, M | Carry propagation |
| `[ ]` | Sort List | 148 | 🟡 Medium | G, A | Merge Sort on LL |
| `[ ]` | Rotate List | 61 | 🟡 Medium | A, G | Find new tail |
| `[ ]` | Reorder List | 143 | 🟡 Medium | A, G, M | Find mid + Reverse + Merge |
| `[ ]` | Copy List with Random Pointer | 138 | 🟡 Medium | A, G, M | Hash Map / Interweave |
| `[ ]` | LRU Cache | 146 | 🟡 Medium | A, G, M | DLL + Hash Map |
| `[ ]` | Odd Even Linked List | 328 | 🟡 Medium | A, G | Two pointer groups |
| `[ ]` | Swap Nodes in Pairs | 24 | 🟡 Medium | A, G | Recursive / Iterative |
| `[ ]` | Add Two Numbers II | 445 | 🟡 Medium | A, G, M | Stack (reverse digits) |
| `[ ]` | Flatten a Multilevel Doubly Linked List | 430 | 🟡 Medium | A, G | DFS / Stack |

---

## DAY 08 — Linked Lists: Advanced
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Reverse Nodes in k-Group | 25 | 🔴 Hard | A, G, M | Recursive group reversal |
| `[ ]` | Next Greater Node In Linked List | 1019 | 🟡 Medium | A, G | Monotonic Stack |
| `[ ]` | Linked List Components | 817 | 🟡 Medium | A, G | Hash Set |
| `[ ]` | Design Browser History | 1472 | 🟡 Medium | A, Ap | DLL / Array |
| `[ ]` | Insert into a Sorted Circular Linked List | 708 | 🟡 Medium | G | Edge case handling |
| `[ ]` | Design Linked List | 707 | 🟡 Medium | A, G | Implementation |
| `[ ]` | Find the Minimum and Maximum Number of Nodes Between Critical Points | 2058 | 🟡 Medium | A | Traversal |
| `[ ]` | Split Linked List in Parts | 725 | 🟡 Medium | A, G | Math + Traversal |
| `[ ]` | Remove Zero Sum Consecutive Nodes from Linked List | 1171 | 🟡 Medium | G, A | Prefix Sum + Hash |
| `[ ]` | Convert Binary Number in a Linked List to Integer | 1290 | 🟢 Easy | A, G | Bit shift |

---

## DAY 09 — Stacks
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Valid Parentheses | 20 | 🟢 Easy | G, A, M | Stack |
| `[ ]` | Min Stack | 155 | 🟡 Medium | A, G, M | Stack of (val, min) |
| `[ ]` | Evaluate Reverse Polish Notation | 150 | 🟡 Medium | A, G | Stack |
| `[ ]` | Generate Parentheses | 22 | 🟡 Medium | G, A, M | Backtracking |
| `[ ]` | Daily Temperatures | 739 | 🟡 Medium | A, G, M | Monotonic Stack |
| `[ ]` | Car Fleet | 853 | 🟡 Medium | A, G | Monotonic Stack |
| `[ ]` | Largest Rectangle in Histogram | 84 | 🔴 Hard | A, G, M | Monotonic Stack |
| `[ ]` | Decode String | 394 | 🟡 Medium | A, G, M | Stack |
| `[ ]` | Asteroid Collision | 735 | 🟡 Medium | A, G | Stack |
| `[ ]` | Basic Calculator II | 227 | 🟡 Medium | A, G, M | Stack (operators) |

---

## DAY 10 — Queues & Monotonic Structures
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Implement Queue using Stacks | 232 | 🟢 Easy | A, G, M | Two Stacks |
| `[ ]` | Implement Stack using Queues | 225 | 🟢 Easy | A, G | Single Queue rotate |
| `[ ]` | Design Circular Queue | 622 | 🟡 Medium | A, G | Array with pointers |
| `[ ]` | Moving Average from Data Stream | 346 | 🟢 Easy | A, G, M | Queue + running sum |
| `[ ]` | Design Hit Counter | 362 | 🟡 Medium | A, G | Deque / Binary Search |
| `[ ]` | Maximum Width Ramp | 962 | 🟡 Medium | A, G | Monotonic Stack |
| `[ ]` | Online Stock Span | 901 | 🟡 Medium | A, G, M | Monotonic Stack |
| `[ ]` | Next Greater Element I | 496 | 🟢 Easy | A, G | Stack + Hash Map |
| `[ ]` | Next Greater Element II | 503 | 🟡 Medium | A, G | Circular + Monotonic Stack |
| `[ ]` | Number of Recent Calls | 933 | 🟢 Easy | A, G | Queue |

---

## DAY 11 — Binary Search
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Binary Search | 704 | 🟢 Easy | G, A, M | Classic Binary Search |
| `[ ]` | Search a 2D Matrix | 74 | 🟡 Medium | A, G, M | Treat as 1D array |
| `[ ]` | Koko Eating Bananas | 875 | 🟡 Medium | A, G | Binary Search on Answer |
| `[ ]` | Find Minimum in Rotated Sorted Array II | 154 | 🔴 Hard | A, G | Binary Search (duplicates) |
| `[ ]` | Time Based Key-Value Store | 981 | 🟡 Medium | G, A | Binary Search on values |
| `[ ]` | Median of Two Sorted Arrays | 4 | 🔴 Hard | G, A, Ap | Binary Search on partition |
| `[ ]` | Find Peak Element | 162 | 🟡 Medium | G, A, M | Binary Search |
| `[ ]` | Search a 2D Matrix II | 240 | 🟡 Medium | A, G, M | Staircase search |
| `[ ]` | Split Array Largest Sum | 410 | 🔴 Hard | G, A, M | Binary Search on Answer |
| `[ ]` | Capacity to Ship Packages Within D Days | 1011 | 🟡 Medium | A, G | Binary Search on Answer |

---

## DAY 12 — Trees: Traversal & Basics
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Binary Tree Inorder Traversal | 94 | 🟢 Easy | A, G, M | Recursion / Stack |
| `[ ]` | Binary Tree Level Order Traversal | 102 | 🟡 Medium | A, G, M | BFS |
| `[ ]` | Binary Tree Zigzag Level Order Traversal | 103 | 🟡 Medium | A, G, M | BFS + flag |
| `[ ]` | Binary Tree Right Side View | 199 | 🟡 Medium | A, G, M | BFS (last per level) |
| `[ ]` | Maximum Depth of Binary Tree | 104 | 🟢 Easy | A, G, M | DFS |
| `[ ]` | Minimum Depth of Binary Tree | 111 | 🟢 Easy | A, G | BFS (first leaf) |
| `[ ]` | Symmetric Tree | 101 | 🟢 Easy | A, G, M | DFS mirror check |
| `[ ]` | Same Tree | 100 | 🟢 Easy | A, G, M | DFS |
| `[ ]` | Invert Binary Tree | 226 | 🟢 Easy | A, G, M | DFS |
| `[ ]` | Average of Levels in Binary Tree | 637 | 🟢 Easy | A, G | BFS |

---

## DAY 13 — Trees: Properties & Paths
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Diameter of Binary Tree | 543 | 🟢 Easy | A, G, M | DFS (left + right depth) |
| `[ ]` | Balanced Binary Tree | 110 | 🟢 Easy | A, G, M | DFS height check |
| `[ ]` | Path Sum | 112 | 🟢 Easy | A, G, M | DFS |
| `[ ]` | Path Sum II | 113 | 🟡 Medium | A, G, M | DFS + Backtracking |
| `[ ]` | Binary Tree Maximum Path Sum | 124 | 🔴 Hard | A, G, M | DFS (gain calculation) |
| `[ ]` | Sum Root to Leaf Numbers | 129 | 🟡 Medium | A, G | DFS |
| `[ ]` | Lowest Common Ancestor of a Binary Tree | 236 | 🟡 Medium | A, G, M | DFS |
| `[ ]` | Count Good Nodes in Binary Tree | 1448 | 🟡 Medium | A, G | DFS (track max) |
| `[ ]` | Subtree of Another Tree | 572 | 🟢 Easy | A, G, M | DFS + isSameTree |
| `[ ]` | Path In Zigzag Labelled Binary Tree | 1104 | 🟡 Medium | G, A | Math |

---

## DAY 14 — BST (Binary Search Tree)
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Validate Binary Search Tree | 98 | 🟡 Medium | A, G, M | DFS with bounds |
| `[ ]` | Kth Smallest Element in a BST | 230 | 🟡 Medium | A, G, M | Inorder traversal |
| `[ ]` | Lowest Common Ancestor of a BST | 235 | 🟡 Medium | A, G, M | BST property |
| `[ ]` | Convert Sorted Array to BST | 108 | 🟢 Easy | A, G, M | Divide & Conquer |
| `[ ]` | Insert into a BST | 701 | 🟡 Medium | A, G | Recursion |
| `[ ]` | Delete Node in a BST | 450 | 🟡 Medium | A, G, M | Find inorder successor |
| `[ ]` | Range Sum of BST | 938 | 🟢 Easy | A, G | DFS with pruning |
| `[ ]` | BST Iterator | 173 | 🟡 Medium | A, G, M | Stack (controlled inorder) |
| `[ ]` | Recover Binary Search Tree | 99 | 🟡 Medium | A, G | Morris Traversal / DFS |
| `[ ]` | Two Sum IV — Input is a BST | 653 | 🟢 Easy | A, G | BFS/DFS + Hash Set |

---

## DAY 15 — Trees: Advanced
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Serialize and Deserialize Binary Tree | 297 | 🔴 Hard | A, G, M | BFS/DFS encoding |
| `[ ]` | Binary Tree Cameras | 968 | 🔴 Hard | G, A | Greedy + DFS states |
| `[ ]` | Flatten Binary Tree to Linked List | 114 | 🟡 Medium | A, G, M | Morris-like traversal |
| `[ ]` | Construct Binary Tree from Preorder and Inorder | 105 | 🟡 Medium | A, G, M | Divide & Conquer |
| `[ ]` | Construct Binary Tree from Inorder and Postorder | 106 | 🟡 Medium | A, G | Divide & Conquer |
| `[ ]` | Find Duplicate Subtrees | 652 | 🟡 Medium | A, G | DFS + Serialization |
| `[ ]` | All Nodes Distance K in Binary Tree | 863 | 🟡 Medium | A, G, M | BFS from target |
| `[ ]` | Binary Tree Vertical Order Traversal | 314 | 🟡 Medium | A, G, M | BFS + column index |
| `[ ]` | Maximum Width of Binary Tree | 662 | 🟡 Medium | A, G | BFS with indexing |
| `[ ]` | Populating Next Right Pointers in Each Node | 116 | 🟡 Medium | A, G | BFS / O(1) space |

---

## DAY 16 — Heaps: Fundamentals
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Kth Largest Element in an Array | 215 | 🟡 Medium | A, G, M | Min-Heap of size k |
| `[ ]` | Top K Frequent Words | 692 | 🟡 Medium | A, G, M | Heap with custom comparator |
| `[ ]` | Find Median from Data Stream | 295 | 🔴 Hard | A, G, M | Two Heaps (max + min) |
| `[ ]` | K Closest Points to Origin | 973 | 🟡 Medium | A, G, M | Max-Heap / Quickselect |
| `[ ]` | Task Scheduler | 621 | 🟡 Medium | A, G, M | Heap + Greedy |
| `[ ]` | Reorganize String | 767 | 🟡 Medium | A, G, M | Heap + Greedy |
| `[ ]` | Minimum Cost to Connect Sticks | 1167 | 🟡 Medium | A, G | Min-Heap (Huffman) |
| `[ ]` | Furthest Building You Can Reach | 1642 | 🟡 Medium | A, G | Min-Heap (ladder greedy) |
| `[ ]` | Kth Smallest Element in a Sorted Matrix | 378 | 🟡 Medium | A, G, M | Binary Search / Heap |
| `[ ]` | Find K Pairs with Smallest Sums | 373 | 🟡 Medium | A, G | Min-Heap |

---

## DAY 17 — Heaps: Advanced
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | IPO | 502 | 🔴 Hard | A, G | Two Heaps + Greedy |
| `[ ]` | Ugly Number II | 264 | 🟡 Medium | G, A | Three Pointers / Heap |
| `[ ]` | Total Cost to Hire K Workers | 2462 | 🟡 Medium | A, G | Two Heaps |
| `[ ]` | Smallest Number in Infinite Set | 2336 | 🟡 Medium | A, G | Heap + Set |
| `[ ]` | Process Tasks Using Servers | 1882 | 🟡 Medium | A, G | Two Heaps |
| `[ ]` | Minimum Number of Refueling Stops | 871 | 🔴 Hard | A, G | Heap + Greedy |
| `[ ]` | Sort Characters By Frequency | 451 | 🟡 Medium | A, G, M | Heap / Bucket Sort |
| `[ ]` | Rearrange String k Distance Apart | 358 | 🔴 Hard | G, A, M | Heap + Queue |
| `[ ]` | Employee Free Time | 759 | 🔴 Hard | G, A | Heap + Merge |
| `[ ]` | Merge k Sorted Arrays | — | 🟡 Medium | G, A, M | Min-Heap (classic) |

---

## DAY 18 — Graphs: BFS & DFS Basics
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Number of Islands | 200 | 🟡 Medium | A, G, M | BFS / DFS |
| `[ ]` | Clone Graph | 133 | 🟡 Medium | A, G, M | BFS + Hash Map |
| `[ ]` | Max Area of Island | 695 | 🟡 Medium | A, G, M | DFS |
| `[ ]` | Pacific Atlantic Water Flow | 417 | 🟡 Medium | A, G, M | Multi-source BFS/DFS |
| `[ ]` | Surrounded Regions | 130 | 🟡 Medium | A, G, M | BFS from border |
| `[ ]` | Rotting Oranges | 994 | 🟡 Medium | A, G, M | Multi-source BFS |
| `[ ]` | Walls and Gates | 286 | 🟡 Medium | A, G, M | Multi-source BFS |
| `[ ]` | Flood Fill | 733 | 🟢 Easy | A, G, M | DFS |
| `[ ]` | Number of Provinces | 547 | 🟡 Medium | A, G | DFS / Union-Find |
| `[ ]` | Is Graph Bipartite? | 785 | 🟡 Medium | A, G, M | BFS/DFS 2-coloring |

---

## DAY 19 — Graphs: Shortest Path & Topological Sort
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Network Delay Time | 743 | 🟡 Medium | A, G | Dijkstra |
| `[ ]` | Cheapest Flights Within K Stops | 787 | 🟡 Medium | A, G, M | Bellman-Ford / Dijkstra |
| `[ ]` | Path With Minimum Effort | 1631 | 🟡 Medium | A, G | Dijkstra / Binary Search |
| `[ ]` | Course Schedule | 207 | 🟡 Medium | A, G, M | Topological Sort (cycle detect) |
| `[ ]` | Course Schedule II | 210 | 🟡 Medium | A, G, M | Topological Sort |
| `[ ]` | Alien Dictionary | 269 | 🔴 Hard | A, G, M | Topological Sort |
| `[ ]` | Minimum Height Trees | 310 | 🟡 Medium | A, G | Topological trimming |
| `[ ]` | Find Eventual Safe States | 802 | 🟡 Medium | A, G | DFS cycle detection |
| `[ ]` | Reconstruct Itinerary | 332 | 🔴 Hard | A, G, M | Eulerian Path + DFS |
| `[ ]` | All Paths From Source to Target | 797 | 🟡 Medium | A, G, M | DFS |

---

## DAY 20 — Graphs: Union-Find & Advanced
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Redundant Connection | 684 | 🟡 Medium | A, G | Union-Find |
| `[ ]` | Number of Connected Components | 323 | 🟡 Medium | A, G, M | Union-Find / DFS |
| `[ ]` | Graph Valid Tree | 261 | 🟡 Medium | A, G, M | Union-Find / DFS |
| `[ ]` | Accounts Merge | 721 | 🟡 Medium | A, G, M | Union-Find |
| `[ ]` | Most Stones Removed with Same Row or Column | 947 | 🟡 Medium | A, G | Union-Find |
| `[ ]` | Min Cost to Connect All Points | 1584 | 🟡 Medium | A, G | Kruskal / Prim |
| `[ ]` | Satisfiability of Equality Equations | 990 | 🟡 Medium | A, G | Union-Find |
| `[ ]` | Swim in Rising Water | 778 | 🔴 Hard | G, A | Dijkstra / Binary Search |
| `[ ]` | Number of Operations to Make Network Connected | 1319 | 🟡 Medium | A, G | Union-Find |
| `[ ]` | Evaluate Division | 399 | 🟡 Medium | G, A, M | Weighted graph DFS |

---

## DAY 21 — Dynamic Programming: 1D Basics
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Climbing Stairs | 70 | 🟢 Easy | A, G, M | DP / Fibonacci |
| `[ ]` | House Robber | 198 | 🟡 Medium | A, G, M | DP (prev2, prev1) |
| `[ ]` | House Robber II | 213 | 🟡 Medium | A, G, M | DP run twice (circular) |
| `[ ]` | Min Cost Climbing Stairs | 746 | 🟢 Easy | A, G | DP |
| `[ ]` | Decode Ways | 91 | 🟡 Medium | A, G, M | DP |
| `[ ]` | Word Break | 139 | 🟡 Medium | A, G, M | DP + Trie |
| `[ ]` | Coin Change | 322 | 🟡 Medium | A, G, M | DP (unbounded knapsack) |
| `[ ]` | Coin Change II | 518 | 🟡 Medium | A, G, M | DP (count ways) |
| `[ ]` | Longest Increasing Subsequence | 300 | 🟡 Medium | A, G, M | DP / Patience Sort |
| `[ ]` | Perfect Squares | 279 | 🟡 Medium | A, G, M | DP / BFS |

---

## DAY 22 — DP: Subsequences & Strings
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Longest Common Subsequence | 1143 | 🟡 Medium | A, G, M | 2D DP |
| `[ ]` | Distinct Subsequences | 115 | 🔴 Hard | A, G, M | 2D DP |
| `[ ]` | Edit Distance | 72 | 🟡 Medium | A, G, M | 2D DP |
| `[ ]` | Delete Operation for Two Strings | 583 | 🟡 Medium | A, G | 2D DP (LCS) |
| `[ ]` | Uncrossed Lines | 1035 | 🟡 Medium | A, G | 2D DP (LCS variant) |
| `[ ]` | Longest Palindromic Subsequence | 516 | 🟡 Medium | A, G, M | 2D DP |
| `[ ]` | Palindromic Substrings | 647 | 🟡 Medium | A, G, M | Expand around center |
| `[ ]` | Longest Palindromic Substring | 5 | 🟡 Medium | A, G, M | Expand around center / Manacher |
| `[ ]` | Regular Expression Matching | 10 | 🔴 Hard | G, A, M | 2D DP |
| `[ ]` | Wildcard Matching | 44 | 🔴 Hard | G, A, M | 2D DP |

---

## DAY 23 — DP: Knapsack Variants
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Partition Equal Subset Sum | 416 | 🟡 Medium | A, G, M | 0/1 Knapsack |
| `[ ]` | Target Sum | 494 | 🟡 Medium | A, G, M | 0/1 Knapsack / DFS |
| `[ ]` | Last Stone Weight II | 1049 | 🟡 Medium | A, G | 0/1 Knapsack |
| `[ ]` | Ones and Zeroes | 474 | 🟡 Medium | A, G | 2D Knapsack |
| `[ ]` | Combination Sum IV | 377 | 🟡 Medium | A, G, M | DP (permutations) |
| `[ ]` | Minimum Cost For Tickets | 983 | 🟡 Medium | A, G | DP |
| `[ ]` | Integer Break | 343 | 🟡 Medium | A, G | DP / Math |
| `[ ]` | Maximal Square | 221 | 🟡 Medium | A, G, M | 2D DP |
| `[ ]` | Count Square Submatrices with All Ones | 1277 | 🟡 Medium | A, G | 2D DP |
| `[ ]` | Profitable Schemes | 879 | 🔴 Hard | G, A | 3D DP (knapsack) |

---

## DAY 24 — DP: Grid & Matrix
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Unique Paths | 62 | 🟡 Medium | A, G, M | 2D DP |
| `[ ]` | Unique Paths II | 63 | 🟡 Medium | A, G, M | 2D DP (with obstacles) |
| `[ ]` | Minimum Path Sum | 64 | 🟡 Medium | A, G, M | 2D DP |
| `[ ]` | Triangle | 120 | 🟡 Medium | A, G, M | DP bottom-up |
| `[ ]` | Dungeon Game | 174 | 🔴 Hard | A, G | DP reverse |
| `[ ]` | Paint House | 256 | 🟡 Medium | A, G, M | DP |
| `[ ]` | Paint House II | 265 | 🔴 Hard | A, G, M | DP O(nk) optimized |
| `[ ]` | Cherry Pickup II | 1463 | 🔴 Hard | G, A | 3D DP |
| `[ ]` | Maximum Points You Can Obtain from Cards | 1423 | 🟡 Medium | A, G | Sliding Window |
| `[ ]` | Maximal Rectangle | 85 | 🔴 Hard | A, G, M | Stack (histogram) |

---

## DAY 25 — DP: Stocks & Interval DP
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Best Time to Buy and Sell Stock II | 122 | 🟡 Medium | A, G, M | Greedy |
| `[ ]` | Best Time to Buy and Sell Stock III | 123 | 🔴 Hard | A, G | DP State Machine |
| `[ ]` | Best Time to Buy and Sell Stock IV | 188 | 🔴 Hard | A, G | DP State Machine |
| `[ ]` | Best Time to Buy and Sell Stock with Cooldown | 309 | 🟡 Medium | A, G, M | DP (hold/sell/rest) |
| `[ ]` | Best Time to Buy and Sell Stock with Transaction Fee | 714 | 🟡 Medium | A, G | DP |
| `[ ]` | Burst Balloons | 312 | 🔴 Hard | G, A | Interval DP |
| `[ ]` | Strange Printer | 664 | 🔴 Hard | G, A | Interval DP |
| `[ ]` | Minimum Cost to Cut a Stick | 1547 | 🔴 Hard | A, G | Interval DP |
| `[ ]` | Longest Arithmetic Subsequence | 1027 | 🟡 Medium | A, G | DP with Hash Map |
| `[ ]` | Stone Game | 877 | 🟡 Medium | A, G | DP / Math |

---

## DAY 26 — Backtracking: Combinations & Permutations
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Subsets | 78 | 🟡 Medium | A, G, M | Backtracking |
| `[ ]` | Subsets II | 90 | 🟡 Medium | A, G, M | Backtracking (dedup sort) |
| `[ ]` | Permutations | 46 | 🟡 Medium | A, G, M | Backtracking |
| `[ ]` | Permutations II | 47 | 🟡 Medium | A, G, M | Backtracking (dedup) |
| `[ ]` | Combination Sum | 39 | 🟡 Medium | A, G, M | Backtracking (reuse) |
| `[ ]` | Combination Sum II | 40 | 🟡 Medium | A, G, M | Backtracking (dedup) |
| `[ ]` | Combination Sum III | 216 | 🟡 Medium | A, G, M | Backtracking (k items) |
| `[ ]` | Letter Combinations of a Phone Number | 17 | 🟡 Medium | A, G, M | Backtracking |
| `[ ]` | Palindrome Partitioning | 131 | 🟡 Medium | A, G, M | Backtracking + DP |
| `[ ]` | Restore IP Addresses | 93 | 🟡 Medium | A, G, M | Backtracking |

---

## DAY 27 — Backtracking: Constraint Problems
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | N-Queens | 51 | 🔴 Hard | A, G, M | Backtracking |
| `[ ]` | N-Queens II | 52 | 🔴 Hard | A, G | Backtracking (count) |
| `[ ]` | Sudoku Solver | 37 | 🔴 Hard | A, G, M | Backtracking |
| `[ ]` | Word Search | 79 | 🟡 Medium | A, G, M | DFS + Backtracking |
| `[ ]` | Word Search II | 212 | 🔴 Hard | A, G, M | Trie + Backtracking |
| `[ ]` | Expression Add Operators | 282 | 🔴 Hard | A, G, M | Backtracking |
| `[ ]` | Remove Invalid Parentheses | 301 | 🔴 Hard | A, G, M | BFS / Backtracking |
| `[ ]` | Beautiful Arrangement | 526 | 🟡 Medium | A, G | Backtracking |
| `[ ]` | 24 Game | 679 | 🔴 Hard | G, A | Backtracking (permute ops) |
| `[ ]` | Matchsticks to Square | 473 | 🟡 Medium | A, G | Backtracking + pruning |

---

## DAY 28 — Tries
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Implement Trie (Prefix Tree) | 208 | 🟡 Medium | A, G, M | Trie implementation |
| `[ ]` | Design Add and Search Words Data Structure | 211 | 🟡 Medium | A, G, M | Trie + DFS (wildcard) |
| `[ ]` | Replace Words | 648 | 🟡 Medium | A, G, M | Trie lookup |
| `[ ]` | Maximum XOR of Two Numbers in an Array | 421 | 🟡 Medium | A, G, M | Bit Trie |
| `[ ]` | Concatenated Words | 472 | 🔴 Hard | A, G, M | Trie + DP |
| `[ ]` | Search Suggestions System | 1268 | 🟡 Medium | A, G | Trie / Binary Search |
| `[ ]` | Design In-Memory File System | 588 | 🔴 Hard | A, G | Trie |
| `[ ]` | Prefix and Suffix Search | 745 | 🔴 Hard | G, A | Trie (wrapped words) |
| `[ ]` | Stream of Characters | 1032 | 🔴 Hard | A, G | Trie + DFS |
| `[ ]` | Longest Word in Dictionary | 720 | 🟡 Medium | G, A, M | Trie / Hash Set |

---

## DAY 29 — Bit Manipulation
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Single Number | 136 | 🟢 Easy | A, G, M | XOR |
| `[ ]` | Single Number II | 137 | 🟡 Medium | A, G, M | Bit counting (mod 3) |
| `[ ]` | Single Number III | 260 | 🟡 Medium | A, G | XOR + grouping |
| `[ ]` | Number of 1 Bits | 191 | 🟢 Easy | A, G, M | n & (n-1) |
| `[ ]` | Counting Bits | 338 | 🟢 Easy | A, G, M | DP + last bit |
| `[ ]` | Reverse Bits | 190 | 🟢 Easy | A, G, M | Bit shift |
| `[ ]` | Missing Number | 268 | 🟢 Easy | A, G, M | XOR / Gauss |
| `[ ]` | Power of Two | 231 | 🟢 Easy | A, G, M | n & (n-1) == 0 |
| `[ ]` | Sum of Two Integers | 371 | 🟡 Medium | A, G, M | Bit addition (carry) |
| `[ ]` | Maximum Product of Word Lengths | 318 | 🟡 Medium | A, G | Bitmask for char set |

---

## DAY 30 — Math, Intervals & Mixed Review
| # | Problem | LC# | Diff | Companies | Key Technique |
|---|---------|-----|------|-----------|---------------|
| `[ ]` | Count Primes | 204 | 🟡 Medium | A, G, M | Sieve of Eratosthenes |
| `[ ]` | Pow(x, n) | 50 | 🟡 Medium | A, G, M | Fast Exponentiation |
| `[ ]` | Sqrt(x) | 69 | 🟢 Easy | A, G, M | Binary Search |
| `[ ]` | Roman to Integer | 13 | 🟢 Easy | A, G, M | Greedy scan |
| `[ ]` | Integer to Roman | 12 | 🟡 Medium | A, G, M | Greedy |
| `[ ]` | Factorial Trailing Zeroes | 172 | 🟡 Medium | A, G, M | Count 5s |
| `[ ]` | Happy Number | 202 | 🟢 Easy | A, G, M | Cycle detection |
| `[ ]` | Multiply Strings | 43 | 🟡 Medium | A, G, M | Grade-school multiply |
| `[ ]` | Spiral Matrix II | 59 | 🟡 Medium | A, G | Simulation |
| `[ ]` | Summary Ranges | 228 | 🟢 Easy | G, A, M | Linear scan |

---

## Quick Reference — Patterns Cheat Sheet

| Pattern | Recognise When | Template Idea |
|---------|---------------|---------------|
| **Sliding Window** | Contiguous subarray/substring, optimal size | Expand right, shrink left |
| **Two Pointers** | Sorted array, find pair/triplet, reverse | Start at both ends or same dir |
| **Fast/Slow Pointer** | Linked list cycle, find middle | Slow +1, Fast +2 |
| **Binary Search on Answer** | "Minimum max" or "Maximum min" | lo/hi on answer space |
| **Monotonic Stack** | Next greater/smaller element | Push index, pop when condition met |
| **BFS on Graph** | Shortest path, level-by-level | Deque + visited set |
| **DFS + Backtracking** | All subsets/permutations, constraint solving | Choose → Explore → Unchoose |
| **Topological Sort** | Dependency ordering, cycle in directed graph | BFS Kahn's or DFS post-order |
| **Union-Find** | Connected components, dynamic connectivity | find() + union() with rank/path |
| **Kadane's** | Max subarray, max product | Track local max + global max |
| **Prefix Sum** | Subarray sum queries | prefix[i] = prefix[i-1] + arr[i] |
| **Interval DP** | Merge/split ranges optimally | dp[i][j] over subranges |
| **0/1 Knapsack** | Pick/don't pick, meet capacity/target | dp[i][w] iterate backwards |
| **State Machine DP** | Stock buy/sell, states with transitions | Define states explicitly |