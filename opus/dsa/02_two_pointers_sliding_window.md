# Two Pointers & Sliding Window — FAANG Questions

## Core Concepts

### Two Pointers
Use when the array is **sorted** or you need to find **pairs/triplets** satisfying a condition. Pointers move toward each other or in the same direction.

### Sliding Window
Use for **contiguous subarray/substring** problems where you need to find optimal window satisfying constraints.

**When to use which:**
```
Sorted array + find pair       → Two Pointers (opposite ends)
Partitioning / Dutch flag      → Two/Three Pointers (same direction)
Longest/shortest subarray      → Sliding Window (expand/shrink)
Fixed-size window              → Fixed Sliding Window
String with constraint         → Sliding Window + Hash Map
```

---

## Questions

### 🟢 Easy

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 1 | Valid Palindrome | M, A, G | Two pointers from ends, skip non-alphanumeric |
| 2 | Squares of a Sorted Array | A, G, M | Two pointers from ends, fill result backwards |
| 3 | Remove Element | A, MS | Two pointers same direction |
| 4 | Is Subsequence | A, G, M | Two pointers (one per string) |
| 5 | Merge Sorted Array | M, A, G | Three pointers from end |
| 6 | Reverse String | A, M, MS | Two pointers swap |
| 7 | Maximum Average Subarray I | A, G | Fixed-size sliding window |
| 8 | Longest Harmonious Subsequence | A, G | Sliding window on sorted |

---

### 🟡 Medium

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 9 | 3Sum | M, G, A, MS | Sort + fix one + two pointer |
| 10 | 3Sum Closest | A, G, MS | Sort + two pointer, track min diff |
| 11 | Container With Most Water | A, G, M | Two pointers, move shorter side |
| 12 | Longest Substring Without Repeating Chars | A, M, G, AP | Sliding window + hash set |
| 13 | Longest Repeating Character Replacement | G, A, M | Sliding window + max freq tracking |
| 14 | Permutation in String | M, A, G | Fixed window + frequency match |
| 15 | Minimum Size Subarray Sum | M, A, G | Variable window, shrink when sum ≥ target |
| 16 | Fruit Into Baskets | A, G | Sliding window (at most 2 distinct) |
| 17 | Max Consecutive Ones III | G, A, M | Sliding window (at most k flips) |
| 18 | Subarray Product Less Than K | A, G | Sliding window, shrink when product ≥ k |
| 19 | Sort Colors | M, A, G | Dutch National Flag (3 pointers) |
| 20 | Boats to Save People | G, A, M | Sort + two pointers (greedy pair) |
| 21 | 4Sum | A, G, MS | Sort + nested two pointer |
| 22 | Remove Nth Node From End of List | A, M, G | Two pointers with gap |
| 23 | Trapping Rain Water (two pointer approach) | G, A, M | Two pointers + left_max, right_max |
| 24 | Find All Anagrams in String | M, A, G | Sliding window + frequency array |
| 25 | Grumpy Bookstore Owner | A, G | Fixed window on grumpy minutes |
| 26 | Maximum Points from Cards | A, G, M | Sliding window on complement |
| 27 | Get Equal Substrings Within Budget | G, A | Sliding window on cost |
| 28 | Number of Subsequences That Satisfy Sum | A, G | Sort + two pointers + power of 2 |

---

### 🔴 Hard

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 29 | Minimum Window Substring | M, A, G | Variable window + frequency + valid count |
| 30 | Sliding Window Maximum | A, G, M | Monotonic deque |
| 31 | Substring with Concatenation of All Words | A, G | Sliding window (word-level) |
| 32 | Subarrays with K Different Integers | G, A, M | At most K - At most (K-1) trick |
| 33 | Longest Substring with At Most K Distinct | G, A, M | Sliding window + hash map |
| 34 | Minimum Window Subsequence | G, M | Two pointer DP |
| 35 | Count Subarrays Where Max Element Appears at Least K Times | G, A | Sliding window |

---

## Pattern Recognition Guide

```
┌─────────────────────────────────────────────────────────────┐
│  "Find pair in sorted array"      → Two Pointers (←  →)    │
│  "Longest substring with..."      → Sliding Window          │
│  "Minimum window containing..."   → Sliding Window (shrink) │
│  "Subarray sum ≥ target"          → Sliding Window          │
│  "Palindrome check"               → Two Pointers            │
│  "Merge two sorted"               → Two Pointers            │
│  "Remove duplicates in-place"     → Slow/Fast Pointer       │
│  "At most K distinct"             → Sliding Window + Map    │
│  "Fixed size window max/min"      → Deque                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Sliding Window Template (Python)

```python
def sliding_window(s, k):
    """Variable-size window template"""
    window = {}  # or Counter
    left = 0
    result = 0
    
    for right in range(len(s)):
        # Expand: add s[right] to window
        window[s[right]] = window.get(s[right], 0) + 1
        
        # Shrink: while window is invalid
        while window_is_invalid(window, k):
            window[s[left]] -= 1
            if window[s[left]] == 0:
                del window[s[left]]
            left += 1
        
        # Update result
        result = max(result, right - left + 1)
    
    return result
```

## Two Pointer Template (Python)

```python
def two_pointer(nums, target):
    """Find pair summing to target in sorted array"""
    left, right = 0, len(nums) - 1
    
    while left < right:
        curr_sum = nums[left] + nums[right]
        if curr_sum == target:
            return [left, right]
        elif curr_sum < target:
            left += 1
        else:
            right -= 1
    
    return [-1, -1]
```
