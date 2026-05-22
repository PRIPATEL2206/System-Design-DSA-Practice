# Arrays & Hashing — FAANG Questions

## Core Concept
Arrays are the foundation. Most problems reduce to: **finding patterns using extra space (hash maps)** to avoid brute-force nested loops.

**Key Patterns:**
- Hash Map for O(1) lookup
- Frequency counting
- Prefix sums
- In-place modification (when space is constrained)

---

## Questions

### 🟢 Easy (Warm-up — Solve in < 10 min)

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 1 | Two Sum | G, A, M, AP, MS | Hash map: complement lookup |
| 2 | Contains Duplicate | A, G, MS | Set for O(1) lookup |
| 3 | Valid Anagram | A, M, G | Frequency array or hash map |
| 4 | Best Time to Buy and Sell Stock | A, M, G, MS | Track min price, max profit |
| 5 | Majority Element | G, A, M | Boyer-Moore Voting |
| 6 | Move Zeroes | M, A, AP | Two pointer swap |
| 7 | Plus One | G, A | Carry propagation |
| 8 | Single Number | A, M, G | XOR all elements |
| 9 | Intersection of Two Arrays II | M, A, G | Hash map count |
| 10 | Remove Duplicates from Sorted Array | M, A, MS | Two pointer in-place |

---

### 🟡 Medium (Interview Level — Solve in < 25 min)

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 11 | Group Anagrams | M, A, G | Sorted string as key |
| 12 | Top K Frequent Elements | A, M, G | Bucket sort or heap |
| 13 | Product of Array Except Self | A, M, G, AP | Prefix and suffix products |
| 14 | Longest Consecutive Sequence | G, A, M | HashSet + sequence start detection |
| 15 | Subarray Sum Equals K | M, G, A | Prefix sum + hash map |
| 16 | 3Sum | M, G, A, MS | Sort + two pointer |
| 17 | 4Sum | A, G, MS | Sort + two pointer (nested) |
| 18 | Container With Most Water | A, G, M | Two pointer from ends |
| 19 | Set Matrix Zeroes | A, M, MS | Use first row/col as markers |
| 20 | Spiral Matrix | A, M, G, MS | Layer-by-layer traversal |
| 21 | Rotate Image | A, M, G | Transpose + reverse rows |
| 22 | Valid Sudoku | A, G, MS | HashSet per row/col/box |
| 23 | Encode and Decode Strings | M, G | Length prefix encoding |
| 24 | Longest Substring Without Repeating Characters | A, M, G, AP | Sliding window + hash set |
| 25 | Minimum Size Subarray Sum | M, A, G | Sliding window |
| 26 | Find All Duplicates in Array | A, M, G | Index marking (negate) |
| 27 | Next Permutation | G, M, A | Find rightmost ascent, swap, reverse |
| 28 | Sort Colors (Dutch National Flag) | M, A, G | Three-way partition |
| 29 | Insert Delete GetRandom O(1) | M, A, G | Array + HashMap |
| 30 | Brick Wall | M, G | Prefix sum frequency |

---

### 🔴 Hard (Senior Level)

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 31 | First Missing Positive | A, G, M | Cyclic sort / index mapping |
| 32 | Trapping Rain Water | G, A, M, AP | Two pointer or stack |
| 33 | Median of Two Sorted Arrays | G, A, M, MS | Binary search on partition |
| 34 | Sliding Window Maximum | A, G, M | Monotonic deque |
| 35 | Minimum Window Substring | M, A, G | Sliding window + frequency |
| 36 | Largest Rectangle in Histogram | G, A, M | Monotonic stack |
| 37 | Count of Smaller Numbers After Self | G, A | Merge sort or BIT |
| 38 | Longest Increasing Subsequence (follow-up O(n log n)) | G, A, M | Patience sorting / binary search |
| 39 | Maximum Subarray (Kadane's + divide & conquer) | A, M, G | Kadane's algorithm |
| 40 | Subarray with given XOR | G, A | Prefix XOR + hash map |

---

## Patterns Cheat Sheet

```
Problem Type          →  Technique
──────────────────────────────────────────────────
Find pair/triplet     →  Two Pointer (sorted) or Hash Map
Subarray sum          →  Prefix Sum + Hash Map
Contiguous subarray   →  Sliding Window
Frequency/counting    →  Hash Map or Array[26]
In-place rearrange    →  Cyclic Sort or Index marking
Range queries         →  Prefix Sum Array
Top K elements        →  Heap or Bucket Sort
```

---

## Solution Template (Python)

```python
def solve(nums):
    # 1. Edge cases
    if not nums:
        return ...
    
    # 2. Data structures
    seen = {}  # or set()
    
    # 3. Single pass
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    
    # 4. Return default
    return -1
```

---

## Daily Practice (Pick 10)
- Day 1: Problems 1-10 (Easy warm-up)
- Day 2: Problems 11-20 (Medium core)
- Day 3: Problems 21-30 (Medium advanced)
- Day 4: Problems 31-40 (Hard stretch) + retry ❌ from Day 1-3
