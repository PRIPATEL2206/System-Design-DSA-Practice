# Trees & BST — FAANG Questions

## Core Concepts

Trees are the **#1 most asked topic** at FAANG interviews. Master these traversals and patterns:

```
Traversal Types:
─────────────────
Inorder (L-Root-R)     → BST gives sorted order
Preorder (Root-L-R)    → Serialize/clone tree
Postorder (L-R-Root)   → Delete/compute from leaves up
Level-order (BFS)      → Level-by-level processing

Key Patterns:
─────────────
DFS (recursive/stack)  → Most tree problems
BFS (queue)            → Level-order, shortest path
Divide & Conquer       → Split at root, combine results
Path problems          → Track sum/path from root or any node
BST property           → Left < Root < Right (inorder = sorted)
```

---

## Questions

### 🟢 Easy

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 1 | Maximum Depth of Binary Tree | A, M, G | DFS: 1 + max(left, right) |
| 2 | Invert Binary Tree | G, A, M | Swap left/right recursively |
| 3 | Same Tree | A, M, G | Compare root + recurse both sides |
| 4 | Symmetric Tree | A, M, G | Mirror comparison (left.left vs right.right) |
| 5 | Subtree of Another Tree | A, M, G | Check isSame at each node |
| 6 | Diameter of Binary Tree | M, G, A | Track max(leftH + rightH) globally |
| 7 | Balanced Binary Tree | A, G, M | Height check returning -1 for unbalanced |
| 8 | Minimum Depth of Binary Tree | A, M | BFS (first leaf) or DFS |
| 9 | Path Sum | A, M, G | DFS: subtract root.val, check leaf |
| 10 | Convert Sorted Array to BST | A, G, M | Binary search: mid as root |
| 11 | Lowest Common Ancestor of BST | A, M, G | Use BST property to decide direction |
| 12 | Search in BST | A, M | Binary search left/right |

---

### 🟡 Medium

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 13 | Binary Tree Level Order Traversal | A, M, G, MS | BFS with level size tracking |
| 14 | Binary Tree Right Side View | A, M, G | BFS: last node per level |
| 15 | Validate BST | A, M, G, MS | Inorder must be strictly increasing |
| 16 | Kth Smallest Element in BST | A, M, G | Inorder traversal, count K |
| 17 | Lowest Common Ancestor of Binary Tree | A, M, G, MS | If found in both subtrees → root is LCA |
| 18 | Construct Binary Tree from Preorder & Inorder | A, M, G | Root from preorder, split inorder |
| 19 | Binary Tree Zigzag Level Order | A, M, G | BFS + alternate direction |
| 20 | Path Sum II (all root-to-leaf paths) | A, M, G | DFS + backtracking |
| 21 | Path Sum III (any start/end) | M, A, G | Prefix sum on paths |
| 22 | Flatten Binary Tree to Linked List | A, M, G | Preorder: connect right chain |
| 23 | Populating Next Right Pointers | A, M, G | BFS or O(1) space with next pointers |
| 24 | Count Good Nodes in Binary Tree | A, G, M | DFS tracking max on path |
| 25 | House Robber III | A, G | DFS: return (rob, not_rob) pair |
| 26 | All Nodes Distance K | A, M, G | Convert to graph → BFS from target |
| 27 | Binary Tree Maximum Path Sum | A, M, G | DFS: track global max, return single path |
| 28 | Delete Node in BST | A, G, M | Find successor, replace value |
| 29 | Serialize and Deserialize BST | A, M, G | Preorder + bounds for deserialize |
| 30 | Sum Root to Leaf Numbers | A, M, G | DFS passing current number |
| 31 | Vertical Order Traversal | A, M, G | BFS + column tracking |
| 32 | Boundary of Binary Tree | A, M | Left boundary + leaves + right boundary |
| 33 | BST Iterator | A, M, G | Controlled inorder with stack |
| 34 | Recover BST (two swapped nodes) | A, G | Inorder: find two violations |
| 35 | Time Needed to Inform All Employees | G, A, M | DFS/BFS on tree |

---

### 🔴 Hard

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 36 | Serialize and Deserialize Binary Tree | A, M, G | Preorder with NULL markers |
| 37 | Binary Tree Maximum Path Sum | A, M, G | DFS: local vs global max |
| 38 | Binary Tree Cameras | G, A | Greedy DFS from leaves |
| 39 | Vertical Order (with strict ordering) | G, M | TreeMap + PriorityQueue |
| 40 | Count Complete Tree Nodes (O(log²n)) | G, A | Binary search on last level |

---

## Template: DFS on Binary Tree

```python
def dfs(root):
    if not root:
        return base_case
    
    left = dfs(root.left)
    right = dfs(root.right)
    
    # Process current node using left, right results
    return combine(root.val, left, right)
```

## Template: BFS Level Order

```python
from collections import deque

def level_order(root):
    if not root:
        return []
    
    result = []
    queue = deque([root])
    
    while queue:
        level_size = len(queue)
        level = []
        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        result.append(level)
    
    return result
```

## Template: Validate BST

```python
def is_valid_bst(root, lo=float('-inf'), hi=float('inf')):
    if not root:
        return True
    if root.val <= lo or root.val >= hi:
        return False
    return (is_valid_bst(root.left, lo, root.val) and
            is_valid_bst(root.right, root.val, hi))
```

## Template: LCA

```python
def lca(root, p, q):
    if not root or root == p or root == q:
        return root
    left = lca(root.left, p, q)
    right = lca(root.right, p, q)
    if left and right:
        return root  # p and q in different subtrees
    return left or right
```
