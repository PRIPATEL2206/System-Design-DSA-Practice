# Linked Lists, Stacks & Queues — Linear Data Structures

## Linked Lists

### Why They Exist
Arrays have O(n) insertion/deletion in the middle. Linked lists have O(1) insertion/deletion if you have a pointer to the node. Trade-off: no random access.

### Core Operations
```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next
```

### The Dummy Head Trick (Use EVERY Time)
```python
# Eliminates edge cases for head modification
def remove_elements(head, val):
    dummy = ListNode(0)
    dummy.next = head
    curr = dummy
    while curr.next:
        if curr.next.val == val:
            curr.next = curr.next.next
        else:
            curr = curr.next
    return dummy.next
```

### Essential Patterns

**Pattern 1: Fast & Slow Pointers (Floyd's)**
```python
# Detect cycle
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return True
    return False

# Find middle of linked list
def find_middle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    return slow  # Middle node
```

**Pattern 2: Reverse a Linked List (Asked EVERYWHERE)**
```python
def reverse_list(head):
    prev = None
    curr = head
    while curr:
        next_node = curr.next
        curr.next = prev
        prev = curr
        curr = next_node
    return prev
```

**Pattern 3: Merge Two Sorted Lists**
```python
def merge_two_lists(l1, l2):
    dummy = ListNode(0)
    curr = dummy
    while l1 and l2:
        if l1.val <= l2.val:
            curr.next = l1
            l1 = l1.next
        else:
            curr.next = l2
            l2 = l2.next
        curr = curr.next
    curr.next = l1 or l2
    return dummy.next
```

**Pattern 4: Remove Nth Node from End**
```python
def remove_nth_from_end(head, n):
    dummy = ListNode(0)
    dummy.next = head
    fast = slow = dummy
    for _ in range(n + 1):
        fast = fast.next
    while fast:
        fast = fast.next
        slow = slow.next
    slow.next = slow.next.next
    return dummy.next
```

### Must-Solve Problems
| Problem | Key Technique |
|---------|--------------|
| Reverse Linked List | Pointer manipulation |
| Merge Two Sorted Lists | Dummy head + compare |
| Linked List Cycle | Fast/slow pointers |
| Remove Nth from End | Two pointers with gap |
| Palindrome Linked List | Find middle + reverse second half |
| Reorder List | Find middle + reverse + merge |
| LRU Cache | Hash map + doubly linked list |
| Add Two Numbers | Carry propagation |

---

## Stacks — LIFO (Last In, First Out)

### Mental Model
Think of a stack of plates. You can only add/remove from the top.

### When to Use a Stack
- **Matching pairs** (parentheses, HTML tags)
- **Nearest greater/smaller element** (monotonic stack)
- **Undo operations** (back button, Ctrl+Z)
- **DFS traversal** (explicit stack instead of recursion)
- **Expression evaluation** (postfix, infix)

### Pattern 1: Valid Parentheses
```python
def is_valid(s):
    stack = []
    mapping = {')': '(', '}': '{', ']': '['}
    for char in s:
        if char in mapping:
            if not stack or stack[-1] != mapping[char]:
                return False
            stack.pop()
        else:
            stack.append(char)
    return len(stack) == 0
```

### Pattern 2: Monotonic Stack (Very Common at FAANG)
**Use when:** Finding the next greater/smaller element for each position.

```python
# Next Greater Element
def next_greater(nums):
    n = len(nums)
    result = [-1] * n
    stack = []  # Stack of indices
    
    for i in range(n):
        while stack and nums[i] > nums[stack[-1]]:
            idx = stack.pop()
            result[idx] = nums[i]
        stack.append(i)
    return result

# Daily Temperatures (LeetCode #739 — asked at Google, Amazon)
def daily_temperatures(temperatures):
    n = len(temperatures)
    result = [0] * n
    stack = []
    
    for i in range(n):
        while stack and temperatures[i] > temperatures[stack[-1]]:
            idx = stack.pop()
            result[idx] = i - idx
        stack.append(i)
    return result
```

### Pattern 3: Min Stack (O(1) getMin)
```python
class MinStack:
    def __init__(self):
        self.stack = []
        self.min_stack = []
    
    def push(self, val):
        self.stack.append(val)
        min_val = min(val, self.min_stack[-1] if self.min_stack else val)
        self.min_stack.append(min_val)
    
    def pop(self):
        self.stack.pop()
        self.min_stack.pop()
    
    def getMin(self):
        return self.min_stack[-1]
```

### Pattern 4: Evaluate Expression
```python
# Basic Calculator (handles +, -, parentheses)
def calculate(s):
    stack = []
    num = 0
    sign = 1
    result = 0
    
    for char in s:
        if char.isdigit():
            num = num * 10 + int(char)
        elif char in '+-':
            result += sign * num
            num = 0
            sign = 1 if char == '+' else -1
        elif char == '(':
            stack.append(result)
            stack.append(sign)
            result = 0
            sign = 1
        elif char == ')':
            result += sign * num
            num = 0
            result *= stack.pop()  # sign
            result += stack.pop()  # previous result
    
    return result + sign * num
```

---

## Queues — FIFO (First In, First Out)

### Mental Model
Think of a line at a restaurant. First person in line gets served first.

### When to Use a Queue
- **BFS traversal** (level-order, shortest path in unweighted graph)
- **Order processing** (first come, first served)
- **Sliding window maximum** (deque)
- **Task scheduling**

### Implementation
```python
from collections import deque

# deque is the proper queue in Python (O(1) on both ends)
queue = deque()
queue.append(1)       # Enqueue
queue.appendleft(0)   # Enqueue at front
queue.popleft()       # Dequeue — O(1)
# NEVER use list.pop(0) — it's O(n)!
```

### Pattern 5: BFS with Queue (Level-Order Traversal)
```python
def level_order(root):
    if not root:
        return []
    result = []
    queue = deque([root])
    while queue:
        level = []
        for _ in range(len(queue)):  # Process entire level
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        result.append(level)
    return result
```

### Pattern 6: Sliding Window Maximum (Monotonic Deque)
```python
# Maximum in every window of size k — asked at Google, Amazon
def max_sliding_window(nums, k):
    dq = deque()  # Stores indices, front is always the max
    result = []
    
    for i in range(len(nums)):
        # Remove elements outside window
        while dq and dq[0] < i - k + 1:
            dq.popleft()
        # Remove smaller elements (they'll never be max)
        while dq and nums[dq[-1]] < nums[i]:
            dq.pop()
        dq.append(i)
        if i >= k - 1:
            result.append(nums[dq[0]])
    return result
```

---

## The LRU Cache — Combines Everything (Asked at Every FAANG)

This is the #1 design problem for coding interviews. It combines hash map + doubly linked list.

```python
class Node:
    def __init__(self, key=0, val=0):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity):
        self.cap = capacity
        self.cache = {}  # key -> Node
        self.head = Node()  # Dummy head (most recent)
        self.tail = Node()  # Dummy tail (least recent)
        self.head.next = self.tail
        self.tail.prev = self.head
    
    def _remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev
    
    def _add_to_front(self, node):
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node
    
    def get(self, key):
        if key in self.cache:
            node = self.cache[key]
            self._remove(node)
            self._add_to_front(node)
            return node.val
        return -1
    
    def put(self, key, value):
        if key in self.cache:
            self._remove(self.cache[key])
        node = Node(key, value)
        self.cache[key] = node
        self._add_to_front(node)
        if len(self.cache) > self.cap:
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]
```

---

## Quick Reference: When to Use What

| Need | Data Structure | Operation |
|------|---------------|-----------|
| Match/nest pairs | Stack | Push open, pop on close |
| Next greater element | Monotonic Stack | Pop smaller, assign |
| Level-by-level traversal | Queue (BFS) | Process level, enqueue children |
| Sliding window max/min | Monotonic Deque | Maintain decreasing/increasing order |
| O(1) get + O(1) put with eviction | Hash Map + DLL (LRU) | Move to front on access |
| Undo/Redo | Two Stacks | Push to undo stack, pop to redo |

---

## Common Mistakes

1. **Using Python list as queue** — `list.pop(0)` is O(n). Use `deque.popleft()`
2. **Forgetting dummy head in linked list** — Makes head operations messy
3. **Not handling single-node linked list** — Always check `head.next is None`
4. **Empty stack pop** — Always check `if stack:` before popping
5. **Monotonic stack direction** — Decreasing stack for "next greater", increasing for "next smaller"
