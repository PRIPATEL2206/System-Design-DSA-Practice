# Binary Search — FAANG Questions

## Core Concepts

Binary search applies whenever the search space has a **monotonic property** — not just sorted arrays!

```
Classic Binary Search:
──────────────────────
Search in sorted array               → Standard BS
Search in rotated sorted array       → Modified BS
Find boundary (first/last true)      → BS on condition

Binary Search on Answer:
────────────────────────
"Minimum value that satisfies X"     → BS on answer space
"Maximum value such that Y"          → BS on answer space
"Split array with min max sum"       → BS on sum

Key Insight:
If you can define a function f(x) that is:
  - False for x < answer
  - True for x >= answer (or vice versa)
Then you can binary search on x!
```

---

## Questions

### 🟢 Easy

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 1 | Binary Search | A, M, G | Standard template |
| 2 | Search Insert Position | A, G, M | Find leftmost position |
| 3 | First Bad Version | A, G, M | BS on condition (isBadVersion) |
| 4 | Sqrt(x) | A, M, G | BS: find largest n where n² ≤ x |
| 5 | Valid Perfect Square | A, G | BS: check if mid² == num |
| 6 | Guess Number Higher or Lower | A, G | Standard BS with API |

---

### 🟡 Medium

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 7 | Search in Rotated Sorted Array | A, M, G, MS | Find sorted half, decide direction |
| 8 | Search in Rotated Sorted Array II (duplicates) | A, M, G | Handle nums[lo] == nums[mid] |
| 9 | Find Minimum in Rotated Sorted Array | A, M, G | BS: compare mid with right |
| 10 | Find Peak Element | A, M, G | BS: go toward higher neighbor |
| 11 | Search a 2D Matrix | A, M, G | Treat as 1D sorted array |
| 12 | Search a 2D Matrix II | A, M, G | Start from top-right corner |
| 13 | Find First and Last Position | A, M, G | Two binary searches (left/right bound) |
| 14 | Koko Eating Bananas | A, G, M | BS on speed: min speed to finish in H hours |
| 15 | Capacity To Ship Packages in D Days | A, G | BS on capacity: min to ship in D days |
| 16 | Split Array Largest Sum | G, A | BS on answer: min max-sum with m splits |
| 17 | Magnetic Force Between Two Balls | G, A | BS on min distance |
| 18 | Minimum Number of Days to Make Bouquets | A, G | BS on days |
| 19 | Single Element in Sorted Array | A, M, G | BS: check pair alignment |
| 20 | Time Based Key-Value Store | A, G, M | BS on timestamps |
| 21 | Random Pick with Weight | M, A, G | Prefix sum + BS |
| 22 | Minimize Max Distance to Gas Station | G | BS on distance |
| 23 | Find K Closest Elements | A, G, M | BS on left boundary of window |
| 24 | Longest Increasing Subsequence (O(n log n)) | A, M, G | BS: patience sort |
| 25 | H-Index II | A, G | BS on sorted citations |

---

### 🔴 Hard

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 26 | Median of Two Sorted Arrays | G, A, M, MS | BS on partition point |
| 27 | Nth Magical Number | G | BS on answer + LCM |
| 28 | Kth Smallest Number in Multiplication Table | G | BS: count ≤ mid in each row |
| 29 | Find in Mountain Array | A, G | Find peak + BS both halves |
| 30 | Count of Range Sum | G, A | Merge sort or BS |
| 31 | Russian Doll Envelopes | G, A | Sort + LIS with BS |
| 32 | Shortest Subarray with Sum at Least K | G, A | Deque (not pure BS but related) |

---

## Templates

### Standard Binary Search
```python
def binary_search(nums, target):
    lo, hi = 0, len(nums) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1
```

### Left Bound (First Occurrence)
```python
def left_bound(nums, target):
    lo, hi = 0, len(nums) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if nums[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return lo  # first index where nums[lo] >= target
```

### Binary Search on Answer
```python
def min_speed_to_eat_bananas(piles, h):
    """Koko Eating Bananas — BS on answer"""
    def can_finish(speed):
        return sum((p + speed - 1) // speed for p in piles) <= h
    
    lo, hi = 1, max(piles)
    while lo < hi:
        mid = (lo + hi) // 2
        if can_finish(mid):
            hi = mid      # try slower
        else:
            lo = mid + 1  # need faster
    return lo
```

### Search in Rotated Array
```python
def search_rotated(nums, target):
    lo, hi = 0, len(nums) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if nums[mid] == target:
            return mid
        
        # Left half is sorted
        if nums[lo] <= nums[mid]:
            if nums[lo] <= target < nums[mid]:
                hi = mid - 1
            else:
                lo = mid + 1
        # Right half is sorted
        else:
            if nums[mid] < target <= nums[hi]:
                lo = mid + 1
            else:
                hi = mid - 1
    return -1
```

### Split Array / Ship Packages Template
```python
def split_array(nums, m):
    """Minimize the largest sum when splitting into m parts"""
    def can_split(max_sum):
        parts = 1
        curr = 0
        for num in nums:
            if curr + num > max_sum:
                parts += 1
                curr = num
            else:
                curr += num
        return parts <= m
    
    lo, hi = max(nums), sum(nums)
    while lo < hi:
        mid = (lo + hi) // 2
        if can_split(mid):
            hi = mid
        else:
            lo = mid + 1
    return lo
```
