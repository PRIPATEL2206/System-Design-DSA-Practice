# Heaps & Priority Queues — Top-K and Scheduling Problems

## Core Concept

A heap is a complete binary tree where every parent is smaller (min-heap) or larger (max-heap) than its children. It gives you O(1) access to the min/max and O(log n) insertion/removal.

**Use a heap when:** You need repeated access to the smallest/largest element.

## Python's heapq (Min-Heap by Default)
```python
import heapq

# Create heap
heap = []
heapq.heappush(heap, 5)
heapq.heappush(heap, 1)
heapq.heappush(heap, 3)

# Pop minimum
smallest = heapq.heappop(heap)  # 1

# Peek without removing
peek = heap[0]

# Heapify existing list — O(n)
arr = [5, 3, 8, 1, 2]
heapq.heapify(arr)

# Max-heap trick: negate values
max_heap = []
heapq.heappush(max_heap, -5)
heapq.heappush(max_heap, -1)
largest = -heapq.heappop(max_heap)  # 5

# Top K elements — O(n log k)
top_k = heapq.nlargest(k, arr)
bottom_k = heapq.nsmallest(k, arr)
```

## Essential Heap Patterns

### Pattern 1: Kth Largest/Smallest
```python
# Kth Largest Element (asked at Amazon, Facebook, Google)
def find_kth_largest(nums, k):
    # Min-heap of size k — top is kth largest
    heap = nums[:k]
    heapq.heapify(heap)
    for num in nums[k:]:
        if num > heap[0]:
            heapq.heapreplace(heap, num)
    return heap[0]

# Time: O(n log k) — better than sorting O(n log n) when k << n
```

### Pattern 2: Merge K Sorted Lists
```python
# Asked at every FAANG company
def merge_k_lists(lists):
    heap = []
    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(heap, (lst.val, i, lst))
    
    dummy = ListNode(0)
    curr = dummy
    while heap:
        val, i, node = heapq.heappop(heap)
        curr.next = node
        curr = curr.next
        if node.next:
            heapq.heappush(heap, (node.next.val, i, node.next))
    
    return dummy.next
# Time: O(N log k) where N = total nodes, k = number of lists
```

### Pattern 3: Top K Frequent Elements
```python
def top_k_frequent(nums, k):
    count = Counter(nums)
    return heapq.nlargest(k, count.keys(), key=count.get)

# Bucket sort approach — O(n)
def top_k_frequent_linear(nums, k):
    count = Counter(nums)
    buckets = [[] for _ in range(len(nums) + 1)]
    for num, freq in count.items():
        buckets[freq].append(num)
    result = []
    for i in range(len(buckets) - 1, -1, -1):
        for num in buckets[i]:
            result.append(num)
            if len(result) == k:
                return result
```

### Pattern 4: Median from Data Stream (Two Heaps)
```python
# Classic hard problem — asked at Amazon, Google, Microsoft
class MedianFinder:
    def __init__(self):
        self.lo = []  # Max-heap (negated) — smaller half
        self.hi = []  # Min-heap — larger half
    
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

### Pattern 5: Task Scheduler
```python
# Task Scheduler with cooldown (asked at Facebook/Meta)
def least_interval(tasks, n):
    counts = Counter(tasks)
    max_heap = [-c for c in counts.values()]
    heapq.heapify(max_heap)
    
    time = 0
    cooldown = deque()  # (count, available_time)
    
    while max_heap or cooldown:
        time += 1
        if max_heap:
            cnt = heapq.heappop(max_heap) + 1  # Execute task
            if cnt != 0:
                cooldown.append((cnt, time + n))
        if cooldown and cooldown[0][1] == time:
            heapq.heappush(max_heap, cooldown.popleft()[0])
    
    return time
```

## When Heap vs. Sort

| Scenario | Heap | Sort |
|----------|------|------|
| Need top K of n elements | O(n log k) | O(n log n) |
| Stream of data (elements arrive over time) | Heap (dynamic) | Can't sort stream |
| Need ALL elements ordered | Sort is simpler | O(n log n) |
| Repeated min/max extraction | O(log n) per | O(n) per without sort |

## Must-Solve Problems
| Problem | Key Idea |
|---------|----------|
| Kth Largest Element | Min-heap of size k |
| Top K Frequent Elements | Heap or bucket sort |
| Merge K Sorted Lists | Min-heap of k heads |
| Find Median from Stream | Two heaps (max + min) |
| Task Scheduler | Max-heap + cooldown queue |
| Meeting Rooms II | Min-heap of end times |
| Reorganize String | Max-heap + greedy |
| K Closest Points to Origin | Max-heap of size k |
