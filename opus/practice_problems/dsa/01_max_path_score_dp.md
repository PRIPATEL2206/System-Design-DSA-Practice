# LeetCode: Maximum Path Score in Grid (DP)

## Problem
Given an m×n grid where each cell is 0, 1, or 2:
- Move only right or down from (0,0) to (m-1, n-1)
- Cell 0: score +0, cost 0
- Cell 1: score +1, cost 1
- Cell 2: score +2, cost 1
- Maximize score with total cost ≤ k
- Return -1 if no valid path exists

---

## My Initial Approach (WRONG)

```python
# dp[i][j] = (max_score, remaining_budget)
# Keep only ONE state per cell — the highest score
```

### Why It Failed
**Single state per cell loses Pareto-optimal paths.**

Example:
```
Grid: [[2, 1, 2],
       [0, 0, 2]]
k = 2

At cell (1,1), two paths arrive:
  Path A: score=3, budget_left=0  ← I kept this (higher score)
  Path B: score=2, budget_left=1  ← I discarded this

Path A can't reach (1,2) — budget=0, but cell (1,2) costs 1
Path B CAN reach (1,2) — budget=1, score becomes 2+2=4

My code returns -1. Correct answer is 4.
```

**Key Insight:** Two paths at the same cell are INCOMPARABLE unless one dominates the other in BOTH score AND budget. You can't just keep max score.

---

## Correct Approach: 3D DP

### State Definition
```
dp[i][j][c] = maximum score achievable at cell (i,j) 
              having spent exactly c total cost to get here
```

### Transition
```
For cell (i,j) with cell_cost and cell_score:
  dp[i][j][c] = max(
      dp[i-1][j][c - cell_cost],   # came from above
      dp[i][j-1][c - cell_cost]    # came from left
  ) + cell_score

  (only if c - cell_cost >= 0 and prev state != -inf)
```

### Answer
```
max(dp[n-1][m-1][0], dp[n-1][m-1][1], ..., dp[n-1][m-1][k])
```

### Why This Works
For each (cell, cost_spent) combination, we track the BEST score independently. No information is lost.

---

## Correct Solution

```python
from typing import List
from math import inf

class Solution:
    def maxPathScore(self, grid: List[List[int]], k: int) -> int:
        n, m = len(grid), len(grid[0])
        # Path length = n+m-1, each cell costs at most 1
        # So max possible cost = n+m-1. Cap k for efficiency.
        k = min(k, n + m - 1)
        NEG_INF = float('-inf')

        # dp[i][j][c] = max score at (i,j) having spent exactly c
        dp = [[[NEG_INF] * (k + 1) for _ in range(m)] for _ in range(n)]

        def cell_cost(i, j):
            return min(1, grid[i][j])  # 0→0, 1→1, 2→1

        # Base case: starting cell
        c0 = cell_cost(0, 0)
        if c0 <= k:
            dp[0][0][c0] = grid[0][0]

        # First row (can only come from left)
        for j in range(1, m):
            c = cell_cost(0, j)
            for spent in range(c, k + 1):
                if dp[0][j-1][spent - c] != NEG_INF:
                    dp[0][j][spent] = max(dp[0][j][spent],
                                          dp[0][j-1][spent - c] + grid[0][j])

        # First column (can only come from above)
        for i in range(1, n):
            c = cell_cost(i, 0)
            for spent in range(c, k + 1):
                if dp[i-1][0][spent - c] != NEG_INF:
                    dp[i][0][spent] = max(dp[i][0][spent],
                                          dp[i-1][0][spent - c] + grid[i][0])

        # Rest of grid
        for i in range(1, n):
            for j in range(1, m):
                c = cell_cost(i, j)
                for spent in range(c, k + 1):
                    best = max(dp[i-1][j][spent - c],    # from above
                               dp[i][j-1][spent - c])    # from left
                    if best != NEG_INF:
                        dp[i][j][spent] = max(dp[i][j][spent],
                                              best + grid[i][j])

        ans = max(dp[n-1][m-1])
        return -1 if ans == NEG_INF else ans
```

---

## Trace Through the Failing Test Case

```
Grid: [[2, 1, 2],
       [0, 0, 2]]
k = 2

After k = min(2, 2+3-1) = 2

Base: cell(0,0) cost=1, dp[0][0][1] = 2

First row:
  j=1: cost=1
    spent=2: dp[0][0][1]=2 → dp[0][1][2] = 2+1 = 3
  j=2: cost=1
    spent=2: dp[0][1][1]=-inf → skip
    (can't reach (0,2) within budget 2: path cost = 1+1+1 = 3 > 2)

First column:
  i=1: cost=0
    spent=0: dp[0][0][0]=-inf → skip
    spent=1: dp[0][0][1]=2 → dp[1][0][1] = 2+0 = 2

Rest:
  (1,1): cost=0
    spent=1: dp[0][1][1]=-inf, dp[1][0][1]=2 → best=2 → dp[1][1][1] = 2+0 = 2
    spent=2: dp[0][1][2]=3, dp[1][0][2]=-inf → best=3 → dp[1][1][2] = 3+0 = 3
    
  (1,2): cost=1
    spent=1: dp[0][2][0]=-inf, dp[1][1][0]=-inf → skip
    spent=2: dp[0][2][1]=-inf, dp[1][1][1]=2 → best=2 → dp[1][2][2] = 2+2 = 4

Answer: max(dp[1][2]) = max(-inf, -inf, 4) = 4 ✓
```

---

## Complexity Analysis

```
Time:  O(n × m × k)
Space: O(n × m × k)

Since k is capped at n+m-1:
  Effective: O(n × m × (n+m))

For a 100×100 grid: 100 × 100 × 199 = ~2M operations → fast
```

---

## Space Optimization (Rolling Array)

Since dp[i][j] only depends on dp[i-1][j] (above) and dp[i][j-1] (left), we can use a single row:

```python
def maxPathScore(self, grid: List[List[int]], k: int) -> int:
    n, m = len(grid), len(grid[0])
    k = min(k, n + m - 1)
    NEG_INF = float('-inf')
    
    # Only need current row
    dp = [[NEG_INF] * (k + 1) for _ in range(m)]
    
    c0 = min(1, grid[0][0])
    if c0 <= k:
        dp[0][c0] = grid[0][0]
    
    # First row
    for j in range(1, m):
        c = min(1, grid[0][j])
        for spent in range(c, k + 1):
            if dp[j-1][spent - c] != NEG_INF:
                dp[j][spent] = max(dp[j][spent], dp[j-1][spent - c] + grid[0][j])
    
    # Remaining rows
    for i in range(1, n):
        new_dp = [[NEG_INF] * (k + 1) for _ in range(m)]
        c = min(1, grid[i][0])
        for spent in range(c, k + 1):
            if dp[0][spent - c] != NEG_INF:
                new_dp[0][spent] = max(new_dp[0][spent], dp[0][spent - c] + grid[i][0])
        
        for j in range(1, m):
            c = min(1, grid[i][j])
            for spent in range(c, k + 1):
                best = max(dp[j][spent - c],        # from above (previous row)
                           new_dp[j-1][spent - c])  # from left (current row)
                if best != NEG_INF:
                    new_dp[j][spent] = max(new_dp[j][spent], best + grid[i][j])
        
        dp = new_dp
    
    ans = max(dp[m-1])
    return -1 if ans == NEG_INF else ans
```

**Space: O(m × k)** instead of O(n × m × k)

---

## Pattern Recognition

This problem is a variant of the classic **"Constrained Grid Path DP"**:

```
Standard grid DP:          dp[i][j] = optimal value at (i,j)
Constrained grid DP:       dp[i][j][resource] = optimal value at (i,j) 
                                                  with resource spent

Similar problems:
  - Minimum Path Sum with k obstacles (LeetCode 1293)
  - Cherry Pickup (LeetCode 741)
  - Grid paths with health constraints
  - Knapsack on a grid
```

### When to add a dimension to DP:
```
Ask: "Can two paths to the same cell be incomparable?"
     "Is there a resource/budget that trades off against the objective?"

If YES → add that resource as a DP dimension.

My mistake: I tried to compress (score, budget) into one state.
Rule: If two quantities trade off against each other, they MUST be separate dimensions.
```
