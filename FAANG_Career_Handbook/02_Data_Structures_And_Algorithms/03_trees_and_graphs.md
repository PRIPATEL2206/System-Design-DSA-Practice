# Trees & Graphs — The Heart of FAANG Interviews

## Trees

### Why Trees Dominate Interviews
Trees test recursion, DFS/BFS, and the ability to think hierarchically. ~25% of FAANG coding questions involve trees.

### Binary Tree Fundamentals
```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
```

### The Three DFS Traversals (Know Cold)
```python
# Inorder: Left → Root → Right (gives sorted order for BST)
def inorder(root):
    if not root:
        return []
    return inorder(root.left) + [root.val] + inorder(root.right)

# Preorder: Root → Left → Right (used for serialization)
def preorder(root):
    if not root:
        return []
    return [root.val] + preorder(root.left) + preorder(root.right)

# Postorder: Left → Right → Root (used for deletion, calculating sizes)
def postorder(root):
    if not root:
        return []
    return postorder(root.left) + postorder(root.right) + [root.val]
```

### Essential Tree Patterns

**Pattern 1: Height/Depth Problems**
```python
# Maximum depth
def max_depth(root):
    if not root:
        return 0
    return 1 + max(max_depth(root.left), max_depth(root.right))

# Check if balanced (every node's subtrees differ by ≤1)
def is_balanced(root):
    def height(node):
        if not node:
            return 0
        left = height(node.left)
        right = height(node.right)
        if left == -1 or right == -1 or abs(left - right) > 1:
            return -1
        return 1 + max(left, right)
    return height(root) != -1
```

**Pattern 2: Path Problems**
```python
# Diameter of binary tree (longest path between any two nodes)
def diameter(root):
    max_d = [0]
    def depth(node):
        if not node:
            return 0
        left = depth(node.left)
        right = depth(node.right)
        max_d[0] = max(max_d[0], left + right)
        return 1 + max(left, right)
    depth(root)
    return max_d[0]

# Maximum path sum (can go through any path — asked at Google)
def max_path_sum(root):
    max_sum = [float('-inf')]
    def gain(node):
        if not node:
            return 0
        left = max(gain(node.left), 0)   # Ignore negative paths
        right = max(gain(node.right), 0)
        max_sum[0] = max(max_sum[0], node.val + left + right)
        return node.val + max(left, right)
    gain(root)
    return max_sum[0]
```

**Pattern 3: BST Properties**
```python
# Validate BST
def is_valid_bst(root, lo=float('-inf'), hi=float('inf')):
    if not root:
        return True
    if root.val <= lo or root.val >= hi:
        return False
    return (is_valid_bst(root.left, lo, root.val) and
            is_valid_bst(root.right, root.val, hi))

# Kth smallest in BST (inorder traversal)
def kth_smallest(root, k):
    stack = []
    curr = root
    while curr or stack:
        while curr:
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()
        k -= 1
        if k == 0:
            return curr.val
        curr = curr.right

# Lowest Common Ancestor (BST version)
def lca_bst(root, p, q):
    while root:
        if p.val < root.val and q.val < root.val:
            root = root.left
        elif p.val > root.val and q.val > root.val:
            root = root.right
        else:
            return root

# LCA (general binary tree)
def lca(root, p, q):
    if not root or root == p or root == q:
        return root
    left = lca(root.left, p, q)
    right = lca(root.right, p, q)
    if left and right:
        return root
    return left or right
```

**Pattern 4: Level-Order + Variants**
```python
# Zigzag level order (asked at Amazon, Microsoft)
def zigzag_order(root):
    if not root:
        return []
    result = []
    queue = deque([root])
    left_to_right = True
    while queue:
        level = []
        for _ in range(len(queue)):
            node = queue.popleft()
            level.append(node.val)
            if node.left: queue.append(node.left)
            if node.right: queue.append(node.right)
        if not left_to_right:
            level.reverse()
        result.append(level)
        left_to_right = not left_to_right
    return result

# Right side view
def right_side_view(root):
    if not root:
        return []
    result = []
    queue = deque([root])
    while queue:
        for i in range(len(queue)):
            node = queue.popleft()
            if i == len(queue):  # Last node in level (after popleft)
                pass
            if node.left: queue.append(node.left)
            if node.right: queue.append(node.right)
        result.append(node.val)  # Last node processed = rightmost
    return result
```

**Pattern 5: Serialize/Deserialize**
```python
# Serialize and deserialize binary tree (asked at every FAANG)
class Codec:
    def serialize(self, root):
        if not root:
            return "null"
        return f"{root.val},{self.serialize(root.left)},{self.serialize(root.right)}"
    
    def deserialize(self, data):
        nodes = iter(data.split(","))
        def build():
            val = next(nodes)
            if val == "null":
                return None
            node = TreeNode(int(val))
            node.left = build()
            node.right = build()
            return node
        return build()
```

---

## Graphs

### Representation
```python
# Adjacency List (most common in interviews)
graph = defaultdict(list)
for u, v in edges:
    graph[u].append(v)
    graph[v].append(u)  # Undirected

# Adjacency Matrix (when you need O(1) edge lookup)
matrix = [[0] * n for _ in range(n)]
matrix[u][v] = 1
```

### DFS vs BFS — When to Use Which

| Use DFS when... | Use BFS when... |
|----------------|-----------------|
| Exploring all paths | Finding shortest path (unweighted) |
| Detecting cycles | Level-by-level processing |
| Topological sort | Minimum steps/moves |
| Connected components | Nearest neighbor |
| Backtracking needed | Spreading outward (rotting oranges) |

### Pattern 6: DFS on Grid (Island Problems)
```python
# Number of Islands (asked at EVERY FAANG company)
def num_islands(grid):
    if not grid:
        return 0
    rows, cols = len(grid), len(grid[0])
    count = 0
    
    def dfs(r, c):
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] == '0':
            return
        grid[r][c] = '0'  # Mark visited
        dfs(r+1, c)
        dfs(r-1, c)
        dfs(r, c+1)
        dfs(r, c-1)
    
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                dfs(r, c)
                count += 1
    return count
```

### Pattern 7: BFS Shortest Path
```python
# Shortest path in unweighted graph
def shortest_path(graph, start, end):
    queue = deque([(start, 0)])
    visited = {start}
    while queue:
        node, dist = queue.popleft()
        if node == end:
            return dist
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append((neighbor, dist + 1))
    return -1

# Rotting Oranges (multi-source BFS — asked at Amazon)
def oranges_rotting(grid):
    rows, cols = len(grid), len(grid[0])
    queue = deque()
    fresh = 0
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == 2:
                queue.append((r, c))
            elif grid[r][c] == 1:
                fresh += 1
    
    if fresh == 0:
        return 0
    
    minutes = 0
    directions = [(0,1),(0,-1),(1,0),(-1,0)]
    while queue:
        for _ in range(len(queue)):
            r, c = queue.popleft()
            for dr, dc in directions:
                nr, nc = r + dr, c + dc
                if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == 1:
                    grid[nr][nc] = 2
                    fresh -= 1
                    queue.append((nr, nc))
        minutes += 1
    
    return minutes - 1 if fresh == 0 else -1
```

### Pattern 8: Topological Sort (DAG problems)
```python
# Course Schedule (detect cycle + ordering — asked at Google, Amazon)
def can_finish(num_courses, prerequisites):
    graph = defaultdict(list)
    in_degree = [0] * num_courses
    
    for course, prereq in prerequisites:
        graph[prereq].append(course)
        in_degree[course] += 1
    
    queue = deque([i for i in range(num_courses) if in_degree[i] == 0])
    completed = 0
    
    while queue:
        node = queue.popleft()
        completed += 1
        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)
    
    return completed == num_courses
```

### Pattern 9: Union-Find (Disjoint Set)
```python
# For connected components, cycle detection in undirected graphs
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n
        self.components = n
    
    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # Path compression
        return self.parent[x]
    
    def union(self, x, y):
        px, py = self.find(x), self.find(y)
        if px == py:
            return False
        if self.rank[px] < self.rank[py]:
            px, py = py, px
        self.parent[py] = px
        if self.rank[px] == self.rank[py]:
            self.rank[px] += 1
        self.components -= 1
        return True
```

### Pattern 10: Dijkstra's (Weighted Shortest Path)
```python
import heapq

def dijkstra(graph, start, n):
    dist = [float('inf')] * n
    dist[start] = 0
    heap = [(0, start)]
    
    while heap:
        d, u = heapq.heappop(heap)
        if d > dist[u]:
            continue
        for v, weight in graph[u]:
            if dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight
                heapq.heappush(heap, (dist[v], v))
    return dist
```

---

## Must-Solve Tree & Graph Problems

| Problem | Pattern | Difficulty |
|---------|---------|-----------|
| Maximum Depth of Binary Tree | Recursion | Easy |
| Invert Binary Tree | Recursion | Easy |
| Validate BST | BST bounds | Medium |
| Level Order Traversal | BFS | Medium |
| Lowest Common Ancestor | DFS | Medium |
| Binary Tree Maximum Path Sum | DFS + global max | Hard |
| Serialize/Deserialize Tree | Preorder + rebuild | Hard |
| Number of Islands | Grid DFS | Medium |
| Clone Graph | BFS/DFS + hash map | Medium |
| Course Schedule | Topological Sort | Medium |
| Word Ladder | BFS | Hard |
| Network Delay Time | Dijkstra | Medium |
| Accounts Merge | Union-Find | Medium |
