# Heap & Priority Queue — FAANG Questions

## Core Concepts

A **heap** gives O(log n) insert/delete and O(1) access to min/max element.

```
Use a Heap when:
────────────────
"Top K" / "Kth largest/smallest"    → Min-heap of size K
"Merge K sorted"                     → Min-heap of K elements
"Continuously find min/max"          → Appropriate heap type
"Median in stream"                   → Two heaps (max + min)
"Scheduling / priority"              → Priority queue

Python: heapq is a MIN-heap
For MAX-heap: negate values → heapq.heappush(h, -val)
```

---

## Questions

### 🟢 Easy

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 1 | Kth Largest Element in a Stream | A, M, G | Min-heap of size K |
| 2 | Last Stone Weight | A, G | Max-heap, smash top two |
| 3 | Relative Ranks | A | Sort with original indices |

---

### 🟡 Medium

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 4 | Kth Largest Element in Array | A, M, G, MS | Quickselect or min-heap of K |
| 5 | Top K Frequent Elements | A, M, G | Heap or bucket sort |
| 6 | Sort Characters By Frequency | A, M, G | Max-heap on frequency |
| 7 | K Closest Points to Origin | A, M, G | Max-heap of K (negate distance) |
| 8 | Task Scheduler | A, M, G | Max-heap + cooldown queue |
| 9 | Reorganize String | A, G, M | Max-heap: place most freq, cooldown |
| 10 | Find K Pairs with Smallest Sums | A, G | Min-heap, expand neighbors |
| 11 | Furthest Building You Can Reach | A, G | Min-heap of ladder uses |
| 12 | Seat Reservation Manager | A | Min-heap of available seats |
| 13 | Design Twitter | A, M | Merge K (each user's tweets) |
| 14 | Ugly Number II | A, G | Min-heap or 3-pointer DP |
| 15 | Maximum Subsequence Score | A, G | Sort + min-heap for top K |
| 16 | Total Cost to Hire K Workers | A, G | Two min-heaps (left/right candidates) |
| 17 | Single-Threaded CPU | A, G | Sort by arrival + min-heap by time |
| 18 | Car Pooling | A, G | Sweep line (or min-heap by drop-off) |
| 19 | Minimum Cost to Connect Sticks | A | Min-heap: always merge two smallest |
| 20 | Process Tasks Using Servers | A, G | Two heaps: available + busy |

---

### 🔴 Hard

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 21 | Find Median from Data Stream | A, M, G, MS | Max-heap (left) + Min-heap (right) |
| 22 | Merge K Sorted Lists | A, M, G | Min-heap of K list heads |
| 23 | Sliding Window Median | G, A | Two heaps + lazy deletion |
| 24 | IPO | A, G | Sort by capital + max-heap profits |
| 25 | Smallest Range Covering Elements from K Lists | G, A | Min-heap + track max |
| 26 | Trapping Rain Water II (3D) | G | Min-heap BFS from borders |
| 27 | Employee Free Time | A, G, M | Merge K sorted interval lists |
| 28 | Minimum Cost to Hire K Workers | G | Sort by wage/quality + max-heap |

---

## Templates

### Top K Elements
```python
import heapq

def top_k_frequent(nums, k):
    freq = Counter(nums)
    # Min-heap of size k
    return heapq.nlargest(k, freq.keys(), key=freq.get)
```

### Kth Largest (Min-Heap of K)
```python
class KthLargest:
    def __init__(self, k, nums):
        self.k = k
        self.heap = nums
        heapq.heapify(self.heap)
        while len(self.heap) > k:
            heapq.heappop(self.heap)
    
    def add(self, val):
        heapq.heappush(self.heap, val)
        if len(self.heap) > self.k:
            heapq.heappop(self.heap)
        return self.heap[0]
```

### Find Median (Two Heaps)
```python
class MedianFinder:
    def __init__(self):
        self.lo = []  # max-heap (negate values)
        self.hi = []  # min-heap
    
    def addNum(self, num):
        heapq.heappush(self.lo, -num)
        heapq.heappush(self.hi, -heapq.heappop(self.lo))
        if len(self.hi) > len(self.lo):
            heapq.heappush(self.lo, -heapq.heappop(self.hi))
    
    def findMedian(self):
        if len(self.lo) > len(self.hi):
            return -self.lo[0]
        return (-self.lo[0] + self.hi[0]) / 2
```

### Merge K Sorted Lists
```python
def merge_k_lists(lists):
    heap = []
    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(heap, (lst.val, i, lst))
    
    dummy = curr = ListNode(0)
    while heap:
        val, i, node = heapq.heappop(heap)
        curr.next = node
        curr = curr.next
        if node.next:
            heapq.heappush(heap, (node.next.val, i, node.next))
    
    return dummy.next
```
