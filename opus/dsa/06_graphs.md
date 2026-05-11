# Graphs — FAANG Questions

## Core Concepts

```
Graph Representations:
──────────────────────
Adjacency List    → dict/list of neighbors (most common)
Adjacency Matrix  → 2D array (dense graphs)
Edge List         → list of (u, v, weight)

Traversal Algorithms:
─────────────────────
BFS               → Shortest path (unweighted), level-order
DFS               → Connected components, cycle detection, topological sort
Dijkstra          → Shortest path (weighted, non-negative)
Bellman-Ford      → Shortest path (negative weights)
Floyd-Warshall    → All-pairs shortest path
Topological Sort  → DAG ordering (course schedule)
Union-Find        → Connected components, cycle detection (undirected)
Kruskal/Prim      → Minimum Spanning Tree

Key Patterns:
─────────────
Grid as graph     → 4-directional neighbors
Multi-source BFS  → Start from all sources simultaneously
Topological Sort  → Dependencies / ordering
Union-Find        → "Are these connected?" queries
Bipartite Check   → 2-coloring with BFS/DFS
```

---

## Questions

### 🟢 Easy

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 1 | Number of Islands | A, M, G, MS | DFS/BFS flood fill |
| 2 | Flood Fill | A, G, M | DFS from starting cell |
| 3 | Find if Path Exists | A, M | BFS/DFS or Union-Find |
| 4 | Find the Town Judge | A, G | Indegree/outdegree |

---

### 🟡 Medium

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 5 | Clone Graph | A, M, G | BFS/DFS + HashMap (old → new) |
| 6 | Course Schedule (can finish?) | A, M, G, MS | Topological sort / cycle detection |
| 7 | Course Schedule II (order) | A, M, G | Topological sort (Kahn's BFS) |
| 8 | Pacific Atlantic Water Flow | A, G, M | BFS from both oceans, find overlap |
| 9 | Number of Connected Components | A, M, G | Union-Find or DFS |
| 10 | Graph Valid Tree | A, M, G | n-1 edges + all connected |
| 11 | Surrounded Regions | A, G, M | BFS from border O's |
| 12 | Rotting Oranges | A, M, G | Multi-source BFS |
| 13 | 01 Matrix | A, G, M | Multi-source BFS from 0s |
| 14 | Walls and Gates | G, A, M | Multi-source BFS from gates |
| 15 | Word Ladder | A, M, G | BFS: each word is a node |
| 16 | Minimum Knight Moves | G, M | BFS on infinite board |
| 17 | Cheapest Flights Within K Stops | A, G, M | Bellman-Ford with K limit or BFS |
| 18 | Network Delay Time | A, G, M | Dijkstra from source |
| 19 | Redundant Connection | A, G | Union-Find: edge creating cycle |
| 20 | Accounts Merge | A, M, G | Union-Find on emails |
| 21 | Evaluate Division | G, A, M | Graph: weighted edge DFS |
| 22 | Is Graph Bipartite? | A, M, G | 2-coloring BFS/DFS |
| 23 | Shortest Path in Binary Matrix | A, M, G | BFS 8-directional |
| 24 | Open the Lock | A, G | BFS on state space |
| 25 | All Paths From Source to Target | A, G, M | DFS/backtracking on DAG |
| 26 | Minimum Height Trees | A, G | Peel leaves layer by layer |
| 27 | Snakes and Ladders | A, G | BFS with board mapping |
| 28 | Number of Provinces | A, M, G | Union-Find or DFS |
| 29 | Shortest Bridge | A, G, M | DFS to find island + BFS to expand |
| 30 | Keys and Rooms | A, G | DFS: collect keys |

---

### 🔴 Hard

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 31 | Word Ladder II (all shortest paths) | A, G, M | BFS for distance + DFS for paths |
| 32 | Alien Dictionary | A, G, M | Topological sort from char ordering |
| 33 | Swim in Rising Water | G, A | Binary search + BFS or Dijkstra |
| 34 | Critical Connections (Bridges) | A, G | Tarjan's bridge-finding |
| 35 | Longest Increasing Path in Matrix | G, A, M | DFS + memoization (topo sort) |
| 36 | Bus Routes | G, A | BFS on routes (not stops) |
| 37 | Shortest Path Visiting All Nodes | G | BFS + bitmask state |
| 38 | Reconstruct Itinerary | G, A, M | Hierholzer's (Eulerian path) |
| 39 | Number of Islands II (online) | G, A | Union-Find with dynamic additions |
| 40 | Making A Large Island | G, A, M | Union-Find + try flipping each 0 |

---

## Templates

### BFS (Shortest Path / Level Order)
```python
from collections import deque

def bfs(graph, start):
    visited = {start}
    queue = deque([start])
    distance = 0
    
    while queue:
        for _ in range(len(queue)):
            node = queue.popleft()
            for neighbor in graph[node]:
                if neighbor not in visited:
                    visited.add(neighbor)
                    queue.append(neighbor)
        distance += 1
    
    return distance
```

### DFS (Connected Components)
```python
def count_components(n, edges):
    graph = defaultdict(list)
    for u, v in edges:
        graph[u].append(v)
        graph[v].append(u)
    
    visited = set()
    count = 0
    
    def dfs(node):
        visited.add(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                dfs(neighbor)
    
    for i in range(n):
        if i not in visited:
            dfs(i)
            count += 1
    return count
```

### Topological Sort (Kahn's BFS)
```python
from collections import deque, defaultdict

def topo_sort(n, prerequisites):
    graph = defaultdict(list)
    indegree = [0] * n
    
    for course, prereq in prerequisites:
        graph[prereq].append(course)
        indegree[course] += 1
    
    queue = deque([i for i in range(n) if indegree[i] == 0])
    order = []
    
    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            indegree[neighbor] -= 1
            if indegree[neighbor] == 0:
                queue.append(neighbor)
    
    return order if len(order) == n else []  # empty = cycle exists
```

### Union-Find
```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n
        self.components = n
    
    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # path compression
        return self.parent[x]
    
    def union(self, x, y):
        px, py = self.find(x), self.find(y)
        if px == py:
            return False  # already connected
        if self.rank[px] < self.rank[py]:
            px, py = py, px
        self.parent[py] = px
        if self.rank[px] == self.rank[py]:
            self.rank[px] += 1
        self.components -= 1
        return True
```

### Dijkstra
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
        for v, w in graph[u]:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                heapq.heappush(heap, (dist[v], v))
    
    return dist
```
