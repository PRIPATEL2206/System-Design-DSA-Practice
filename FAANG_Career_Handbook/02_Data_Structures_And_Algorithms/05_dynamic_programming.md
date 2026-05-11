# Dynamic Programming — The Most Feared Interview Topic

## Why DP Terrifies People (And How to Beat It)

DP isn't magic. It's just **recursion + memory**. If you can write a recursive solution, you can write a DP solution. The secret: learn to recognize the ~10 patterns, not memorize 300 problems.

---

## The DP Framework (Use Every Time)

### Step 1: Can DP solve this?
Two conditions MUST be true:
1. **Optimal substructure** — Optimal solution uses optimal solutions to subproblems
2. **Overlapping subproblems** — Same subproblems computed multiple times

### Step 2: Define the state
What information do I need to describe a subproblem uniquely?
- `dp[i]` = answer considering first i elements
- `dp[i][j]` = answer considering elements i to j, or using capacity j

### Step 3: Find the recurrence
How does the current state relate to previous states?

### Step 4: Identify base cases
What's the answer for the smallest subproblem?

### Step 5: Determine order of computation
Bottom-up: smaller → larger. Ensure dependencies are computed first.

---

## The 10 DP Patterns

### Pattern 1: Linear DP (1D)

**Climbing Stairs / Fibonacci family**
```python
# How many ways to reach step n? (Can take 1 or 2 steps)
def climb_stairs(n):
    if n <= 2:
        return n
    prev2, prev1 = 1, 2
    for i in range(3, n + 1):
        curr = prev1 + prev2
        prev2, prev1 = prev1, curr
    return prev1
# dp[i] = dp[i-1] + dp[i-2]
```

**House Robber (can't rob adjacent houses)**
```python
def rob(nums):
    if len(nums) <= 2:
        return max(nums)
    prev2, prev1 = nums[0], max(nums[0], nums[1])
    for i in range(2, len(nums)):
        curr = max(prev1, prev2 + nums[i])  # Skip or take
        prev2, prev1 = prev1, curr
    return prev1
```

### Pattern 2: 0/1 Knapsack

**For each item: either take it or skip it.**
```python
# Given items with weight and value, maximize value within capacity W
def knapsack(weights, values, W):
    n = len(weights)
    dp = [[0] * (W + 1) for _ in range(n + 1)]
    
    for i in range(1, n + 1):
        for w in range(W + 1):
            dp[i][w] = dp[i-1][w]  # Skip item i
            if weights[i-1] <= w:
                dp[i][w] = max(dp[i][w], dp[i-1][w - weights[i-1]] + values[i-1])
    
    return dp[n][W]

# Space-optimized (1D array, iterate W backwards)
def knapsack_optimized(weights, values, W):
    dp = [0] * (W + 1)
    for i in range(len(weights)):
        for w in range(W, weights[i] - 1, -1):  # Backwards!
            dp[w] = max(dp[w], dp[w - weights[i]] + values[i])
    return dp[W]
```

**Variants:** Subset Sum, Partition Equal Subset, Target Sum, Coin Change (unbounded)

### Pattern 3: Unbounded Knapsack (Coin Change)
```python
# Minimum coins to make amount (each coin usable unlimited times)
def coin_change(coins, amount):
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0
    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i:
                dp[i] = min(dp[i], dp[i - coin] + 1)
    return dp[amount] if dp[amount] != float('inf') else -1

# Number of ways to make amount
def coin_change_ways(coins, amount):
    dp = [0] * (amount + 1)
    dp[0] = 1
    for coin in coins:          # Coin loop OUTSIDE
        for i in range(coin, amount + 1):
            dp[i] += dp[i - coin]
    return dp[amount]
```

### Pattern 4: Longest Common Subsequence (LCS) / String DP
```python
def lcs(text1, text2):
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i-1] == text2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
    
    return dp[m][n]
```

**Variants:** Edit Distance, Longest Common Substring, Longest Palindromic Subsequence

### Pattern 5: Longest Increasing Subsequence (LIS)
```python
# O(n²) approach
def lis(nums):
    n = len(nums)
    dp = [1] * n
    for i in range(1, n):
        for j in range(i):
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)
    return max(dp)

# O(n log n) approach with binary search (patience sorting)
def lis_optimal(nums):
    from bisect import bisect_left
    tails = []
    for num in nums:
        pos = bisect_left(tails, num)
        if pos == len(tails):
            tails.append(num)
        else:
            tails[pos] = num
    return len(tails)
```

### Pattern 6: Grid DP
```python
# Unique Paths (top-left to bottom-right, only move right/down)
def unique_paths(m, n):
    dp = [[1] * n for _ in range(m)]
    for i in range(1, m):
        for j in range(1, n):
            dp[i][j] = dp[i-1][j] + dp[i][j-1]
    return dp[m-1][n-1]

# Minimum Path Sum
def min_path_sum(grid):
    m, n = len(grid), len(grid[0])
    for i in range(m):
        for j in range(n):
            if i == 0 and j == 0:
                continue
            elif i == 0:
                grid[i][j] += grid[i][j-1]
            elif j == 0:
                grid[i][j] += grid[i-1][j]
            else:
                grid[i][j] += min(grid[i-1][j], grid[i][j-1])
    return grid[m-1][n-1]
```

### Pattern 7: Interval DP
```python
# Longest Palindromic Substring
def longest_palindrome(s):
    n = len(s)
    start, max_len = 0, 1
    
    # Expand from center approach (O(n²) time, O(1) space)
    def expand(l, r):
        nonlocal start, max_len
        while l >= 0 and r < n and s[l] == s[r]:
            if r - l + 1 > max_len:
                start, max_len = l, r - l + 1
            l -= 1
            r += 1
    
    for i in range(n):
        expand(i, i)      # Odd length
        expand(i, i + 1)  # Even length
    
    return s[start:start + max_len]
```

### Pattern 8: State Machine DP
```python
# Best Time to Buy and Sell Stock with Cooldown
def max_profit(prices):
    n = len(prices)
    if n < 2:
        return 0
    # States: hold, sold (cooldown), rest
    hold = -prices[0]
    sold = 0
    rest = 0
    for i in range(1, n):
        prev_hold = hold
        hold = max(hold, rest - prices[i])
        rest = max(rest, sold)
        sold = prev_hold + prices[i]
    return max(sold, rest)
```

### Pattern 9: Bitmask DP
```python
# Travelling Salesman Problem (visit all cities, minimize cost)
def tsp(dist, n):
    dp = [[float('inf')] * n for _ in range(1 << n)]
    dp[1][0] = 0  # Start at city 0
    
    for mask in range(1 << n):
        for u in range(n):
            if not (mask & (1 << u)):
                continue
            for v in range(n):
                if mask & (1 << v):
                    continue
                new_mask = mask | (1 << v)
                dp[new_mask][v] = min(dp[new_mask][v], dp[mask][u] + dist[u][v])
    
    full_mask = (1 << n) - 1
    return min(dp[full_mask][i] + dist[i][0] for i in range(1, n))
```

### Pattern 10: DP on Trees
```python
# Maximum path sum in tree, House Robber III
def rob_tree(root):
    def dfs(node):
        if not node:
            return (0, 0)  # (rob_this, skip_this)
        left = dfs(node.left)
        right = dfs(node.right)
        rob = node.val + left[1] + right[1]
        skip = max(left) + max(right)
        return (rob, skip)
    return max(dfs(root))
```

---

## Top-Down vs Bottom-Up

| Top-Down (Memoization) | Bottom-Up (Tabulation) |
|------------------------|----------------------|
| Recursive + cache | Iterative + table |
| Only computes needed states | Computes all states |
| Easier to write (natural recursion) | Better performance (no recursion overhead) |
| Risk of stack overflow | No stack issues |
| Start from answer, work down | Start from base, work up |

```python
# Top-down
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    if n <= 1:
        return n
    return fib(n-1) + fib(n-2)
```

---

## DP Decision Tree (Which Pattern?)

```
Is it about subsequences of two strings?     → LCS / Edit Distance
Is it a min/max on an array with choices?     → Linear DP (Robber, Stairs)
Is it "can you reach target with items"?      → Knapsack / Subset Sum
Is it "minimum cost to traverse grid"?        → Grid DP
Is it "longest increasing/decreasing..."?     → LIS
Is it about buying/selling with states?       → State Machine DP
Does it involve visiting all nodes (small n)? → Bitmask DP
Is it about palindromes?                      → Interval DP / Expand from center
```

---

## Must-Solve Problems (Priority Order)

| # | Problem | Pattern |
|---|---------|---------|
| 1 | Climbing Stairs | Linear |
| 2 | House Robber | Linear |
| 3 | Coin Change | Unbounded Knapsack |
| 4 | Longest Common Subsequence | String DP |
| 5 | Longest Increasing Subsequence | LIS |
| 6 | 0/1 Knapsack | Knapsack |
| 7 | Edit Distance | String DP |
| 8 | Unique Paths | Grid |
| 9 | Word Break | Linear + Hash Set |
| 10 | Partition Equal Subset Sum | Knapsack variant |
| 11 | Longest Palindromic Substring | Interval/Expand |
| 12 | Best Time to Buy/Sell Stock series | State Machine |
| 13 | Decode Ways | Linear |
| 14 | Maximum Product Subarray | Linear (track min & max) |
| 15 | Regular Expression Matching | String DP |

---

## Common Mistakes

1. **Wrong state definition** — If your recurrence doesn't work, your state is missing information
2. **Wrong base case** — Off-by-one errors in initialization
3. **Wrong iteration order** — For 0/1 knapsack with 1D array, iterate capacity BACKWARDS
4. **Coin change ways vs min coins** — Loop order matters! Coins outside = combinations, inside = permutations
5. **Not optimizing space** — If dp[i] only depends on dp[i-1], use two variables instead of array
