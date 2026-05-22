# Linked Lists — FAANG Questions

## Core Concepts

Linked list problems test your ability to **manipulate pointers** without getting confused. The key techniques:

```
Technique                    When to Use
────────────────────────────────────────────────────────
Slow/Fast Pointer           Cycle detection, find middle
Dummy Node                  Simplify head modifications
Reverse (iterative)         Reverse full or partial list
Merge                       Merge sorted lists
Recursion                   Elegant but O(n) stack space
```

**Common Mistakes:**
- Forgetting to handle NULL next pointers
- Losing reference to the head
- Not using a dummy node when head might change

---

## Questions

### 🟢 Easy

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 1 | Reverse Linked List | A, M, G, MS, AP | Three pointers: prev, curr, next |
| 2 | Merge Two Sorted Lists | A, M, G, MS | Dummy node + compare |
| 3 | Linked List Cycle | A, M, G | Fast/slow pointer |
| 4 | Middle of Linked List | A, G, M | Fast/slow pointer |
| 5 | Remove Duplicates from Sorted List | A, M, G | Skip while val == next.val |
| 6 | Intersection of Two Linked Lists | A, M, G | Two pointer length equalization |
| 7 | Palindrome Linked List | A, M, G | Find mid → reverse second half → compare |
| 8 | Remove Linked List Elements | A, M | Dummy node + skip |

---

### 🟡 Medium

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 9 | Add Two Numbers | A, M, G, MS | Traverse both + carry |
| 10 | Remove Nth Node From End | A, M, G | Two pointers with N gap |
| 11 | Reorder List | A, M, G | Find mid → reverse → merge alternately |
| 12 | Linked List Cycle II (find entry) | A, M, G | Floyd's: meet → reset one to head |
| 13 | Copy List with Random Pointer | A, M, G | HashMap clone or interleave trick |
| 14 | Swap Nodes in Pairs | A, M, G | Iterative or recursive swap |
| 15 | Sort List | A, M, G | Merge sort (find mid + merge) |
| 16 | Partition List | A, M | Two dummy lists (< x and >= x) |
| 17 | Rotate List | A, M, G | Make circular → break at n-k |
| 18 | Flatten a Multilevel Doubly Linked List | A, M | Stack or recursion for child |
| 19 | Add Two Numbers II (MSB first) | A, M, G | Stack both → add with carry |
| 20 | Odd Even Linked List | A, M | Separate odd/even indexed nodes |
| 21 | Design Linked List | A, M | Implement from scratch |
| 22 | LRU Cache | A, M, G, MS, AP | HashMap + Doubly Linked List |
| 23 | Delete Node in BST (given node only) | M, A | Copy next value + delete next |
| 24 | Insert into Sorted Circular List | M, G | Handle wrap-around case |

---

### 🔴 Hard

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 25 | Merge K Sorted Lists | A, M, G, MS | Min-heap of K heads |
| 26 | Reverse Nodes in K-Group | A, M, G | Reverse K nodes iteratively |
| 27 | LFU Cache | A, G, M | HashMap + frequency doubly-linked lists |
| 28 | Design Skiplist | G | Multiple levels of linked lists |

---

## Essential Patterns

### Reverse a Linked List (Iterative)
```python
def reverse(head):
    prev = None
    curr = head
    while curr:
        next_node = curr.next
        curr.next = prev
        prev = curr
        curr = next_node
    return prev  # new head
```

### Detect Cycle & Find Entry
```python
def detect_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            # Find entry point
            slow = head
            while slow != fast:
                slow = slow.next
                fast = fast.next
            return slow  # cycle start
    return None
```

### Find Middle Node
```python
def find_middle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    return slow  # middle (right-middle for even)
```

### Merge Two Sorted Lists
```python
def merge(l1, l2):
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

### LRU Cache Structure
```python
class LRUCache:
    def __init__(self, capacity):
        self.cap = capacity
        self.cache = {}  # key → node
        self.head = Node(0, 0)  # dummy head (MRU end)
        self.tail = Node(0, 0)  # dummy tail (LRU end)
        self.head.next = self.tail
        self.tail.prev = self.head
    
    def get(self, key):
        if key in self.cache:
            node = self.cache[key]
            self._remove(node)
            self._add(node)  # move to front
            return node.val
        return -1
    
    def put(self, key, value):
        if key in self.cache:
            self._remove(self.cache[key])
        node = Node(key, value)
        self._add(node)
        self.cache[key] = node
        if len(self.cache) > self.cap:
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]
```
