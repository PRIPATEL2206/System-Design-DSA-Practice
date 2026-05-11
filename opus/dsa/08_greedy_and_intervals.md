# Greedy & Intervals — FAANG Questions

## Core Concepts

**Greedy** = Make the locally optimal choice at each step, hoping it leads to global optimum.

**When greedy works:** Problems with **greedy choice property** (local optimal → global optimal) and **optimal substructure**.

```
Common Greedy Patterns:
───────────────────────
Activity Selection     → Sort by end time, pick non-overlapping
Interval Scheduling    → Sort by start/end, merge or count
Jump problems          → Track farthest reachable
Huffman-style          → Always process smallest first (heap)
Exchange argument      → Prove swapping any two doesn't improve
```

---

## Questions

### 🟢 Easy

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 1 | Assign Cookies | A, G | Sort both, match greed ≤ cookie size |
| 2 | Lemonade Change | A, G | Greedy: use $10 bills before $5 |
| 3 | Best Time to Buy and Sell Stock II | A, M, G | Add every positive difference |
| 4 | Maximum Units on a Truck | A | Sort by units/box desc, fill truck |

---

### 🟡 Medium

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 5 | Jump Game | A, M, G | Track max reachable index |
| 6 | Jump Game II | A, M, G | BFS-like: current end, farthest |
| 7 | Gas Station | A, G, M | If total ≥ 0, start where running sum is lowest |
| 8 | Merge Intervals | A, M, G, MS, AP | Sort by start, merge overlapping |
| 9 | Insert Interval | A, M, G | Find overlap region, merge |
| 10 | Non-overlapping Intervals | A, M, G | Sort by end, count removals |
| 11 | Minimum Number of Arrows to Burst Balloons | A, G, M | Sort by end, count non-overlapping |
| 12 | Meeting Rooms II | A, M, G, MS | Sort + min-heap for end times |
| 13 | Task Scheduler | A, M, G | Most frequent task drives idle slots |
| 14 | Partition Labels | A, M, G | Track last occurrence, extend partition |
| 15 | Reorganize String | A, G, M | Max heap: alternate most frequent |
| 16 | Queue Reconstruction by Height | G, A | Sort desc height, insert by k-index |
| 17 | Minimum Platforms (trains) | A, G | Sort arrive/depart, count overlap |
| 18 | Car Pooling | A, G | Sweep line: +passengers at start, -at end |
| 19 | Interval List Intersections | A, M, G | Two pointers on sorted intervals |
| 20 | Video Stitching | A, G | Greedy extend (jump game variant) |
| 21 | Candy | A, G | Two passes: left-to-right, right-to-left |
| 22 | Boats to Save People | G, A, M | Sort + two pointers |
| 23 | Bag of Tokens | G, A | Sort, buy cheap tokens, sell expensive |
| 24 | Advantage Shuffle | G, A | Sort both, assign greedily (or "waste" smallest) |
| 25 | Minimum Cost to Hire K Workers | G | Sort by wage/quality ratio |

---

### 🔴 Hard

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 26 | Minimum Number of Refueling Stops | G, A | Max-heap of passed stations |
| 27 | IPO | A, G | Sort by capital, max-heap for profit |
| 28 | Employee Free Time | A, G, M | Merge all intervals, find gaps |
| 29 | Course Schedule III | G, A | Sort by deadline, max-heap of durations |
| 30 | Patching Array | G | Greedy: extend reachable range |

---

## Templates

### Merge Intervals
```python
def merge(intervals):
    intervals.sort(key=lambda x: x[0])
    merged = [intervals[0]]
    
    for start, end in intervals[1:]:
        if start <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], end)
        else:
            merged.append([start, end])
    
    return merged
```

### Meeting Rooms II (Min Rooms Needed)
```python
import heapq

def min_meeting_rooms(intervals):
    intervals.sort(key=lambda x: x[0])
    heap = []  # end times
    
    for start, end in intervals:
        if heap and heap[0] <= start:
            heapq.heappop(heap)  # reuse room
        heapq.heappush(heap, end)
    
    return len(heap)
```

### Jump Game II (Greedy BFS)
```python
def jump(nums):
    jumps = 0
    current_end = 0
    farthest = 0
    
    for i in range(len(nums) - 1):
        farthest = max(farthest, i + nums[i])
        if i == current_end:
            jumps += 1
            current_end = farthest
    
    return jumps
```

### Task Scheduler
```python
def least_interval(tasks, n):
    freq = Counter(tasks)
    max_freq = max(freq.values())
    max_count = sum(1 for v in freq.values() if v == max_freq)
    
    # (max_freq - 1) full chunks of (n+1) + last partial chunk
    return max(len(tasks), (max_freq - 1) * (n + 1) + max_count)
```
