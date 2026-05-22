# Dynamic Programming — FAANG Questions

## Core Concepts

DP = **Optimal substructure** + **Overlapping subproblems**

```
When to use DP:
───────────────
"Find minimum/maximum..."
"Count number of ways..."
"Is it possible to..."
"Longest/shortest..."
with choices at each step

DP Categories:
──────────────
1D DP            → Fibonacci, climbing stairs, house robber
2D DP            → Grid paths, LCS, knapsack
Interval DP      → Matrix chain, burst balloons
State Machine    → Stock problems, state transitions
Bitmask DP       → TSP, subset problems
Tree DP          → Max path, house robber III
String DP        → Edit distance, regex match
```

### Approach
1. **Define state** — What does dp[i] represent?
2. **Transition** — How does dp[i] relate to smaller subproblems?
3. **Base case** — What's dp[0] or dp[1]?
4. **Order** — Fill table in correct order
5. **Answer** — Where is the final answer?

---

## Questions

### 🟢 Easy

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 1 | Climbing Stairs | A, M, G | dp[i] = dp[i-1] + dp[i-2] |
| 2 | Min Cost Climbing Stairs | A, G | dp[i] = cost[i] + min(dp[i-1], dp[i-2]) |
| 3 | House Robber | A, M, G | dp[i] = max(dp[i-1], dp[i-2] + nums[i]) |
| 4 | Maximum Subarray (Kadane's) | A, M, G, MS | dp[i] = max(nums[i], dp[i-1] + nums[i]) |
| 5 | Best Time to Buy and Sell Stock | A, M, G | Track min price, max profit |
| 6 | Counting Bits | A, M | dp[i] = dp[i >> 1] + (i & 1) |
| 7 | Divisor Game | A, G | dp[i] = any(not dp[i-x] for x dividing i) |
| 8 | Pascal's Triangle | A, M, G | dp[i][j] = dp[i-1][j-1] + dp[i-1][j] |

---

### 🟡 Medium

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 9 | House Robber II (circular) | A, M, G | Two passes: [0:n-1] and [1:n] |
| 10 | Coin Change | A, M, G, MS | dp[amount] = min(dp[amount-coin] + 1) |
| 11 | Coin Change II (count ways) | A, G, M | dp[amount] += dp[amount-coin] |
| 12 | Longest Increasing Subsequence | A, M, G, MS | dp[i] = max(dp[j]+1) for j < i, nums[j] < nums[i] |
| 13 | Longest Common Subsequence | A, M, G | 2D: dp[i][j] = match or max(skip) |
| 14 | Unique Paths | A, M, G | dp[i][j] = dp[i-1][j] + dp[i][j-1] |
| 15 | Unique Paths II (with obstacles) | A, M, G | Skip cells with obstacles |
| 16 | Minimum Path Sum | A, M, G | dp[i][j] = grid[i][j] + min(top, left) |
| 17 | Decode Ways | A, M, G | dp[i] = dp[i-1] (if valid 1-digit) + dp[i-2] (if valid 2-digit) |
| 18 | Word Break | A, M, G, MS | dp[i] = any(dp[j] and s[j:i] in dict) |
| 19 | Partition Equal Subset Sum | A, M, G | 0/1 knapsack with target = sum/2 |
| 20 | Target Sum | A, M, G | Subset sum variant |
| 21 | Perfect Squares | A, G, M | dp[n] = min(dp[n - i²] + 1) |
| 22 | Longest Palindromic Substring | A, M, G | Expand around center or dp[i][j] |
| 23 | Palindromic Substrings (count) | A, M, G | dp[i][j] = s[i]==s[j] and dp[i+1][j-1] |
| 24 | Maximum Product Subarray | A, M, G | Track max AND min (negatives flip) |
| 25 | Jump Game | A, M, G | Greedy: track farthest reachable |
| 26 | Jump Game II (min jumps) | A, M, G | BFS-like or greedy |
| 27 | Delete and Earn | A, G | House robber on frequency array |
| 28 | Best Time to Buy/Sell Stock with Cooldown | A, M, G | State machine: hold, sold, cooldown |
| 29 | Best Time to Buy/Sell Stock with Fee | A, M, G | State machine: hold, cash |
| 30 | Interleaving String | G, A, M | 2D DP: dp[i][j] = valid interleave |
| 31 | Edit Distance | A, M, G | 2D: insert/delete/replace costs |
| 32 | Ones and Zeroes | A, G | 2D knapsack (0s and 1s as dimensions) |
| 33 | Longest String Chain | A, G, M | Sort by length + LIS variant |
| 34 | Maximal Square | A, M, G | dp[i][j] = min(top, left, diag) + 1 |
| 35 | Combination Sum IV | A, G | dp[target] += dp[target - num] |

---

### 🔴 Hard

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 36 | Longest Valid Parentheses | A, G, M | dp[i] = length ending at i |
| 37 | Regular Expression Matching | G, A, M | 2D DP with '.' and '*' handling |
| 38 | Wildcard Matching | G, A, M | 2D DP with '?' and '*' |
| 39 | Burst Balloons | G, A | Interval DP: last balloon to burst |
| 40 | Word Break II (all sentences) | A, M, G | DP + backtracking |
| 41 | Distinct Subsequences | A, G | 2D: count ways to form t from s |
| 42 | Minimum Cost to Cut a Stick | G, A | Interval DP |
| 43 | Stone Game variations | G, A | Minimax DP |
| 44 | Palindrome Partitioning II | G, A | dp[i] = min cuts for s[0:i] |
| 45 | Frog Jump | A, G | HashMap of (stone → set of jump sizes) |

---

## Templates

### 1D DP (House Robber Pattern)
```python
def rob(nums):
    if len(nums) <= 2:
        return max(nums)
    dp = [0] * len(nums)
    dp[0] = nums[0]
    dp[1] = max(nums[0], nums[1])
    for i in range(2, len(nums)):
        dp[i] = max(dp[i-1], dp[i-2] + nums[i])
    return dp[-1]
```

### 2D DP (LCS Pattern)
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

### Knapsack (0/1)
```python
def knapsack(weights, values, capacity):
    n = len(weights)
    dp = [0] * (capacity + 1)
    for i in range(n):
        for w in range(capacity, weights[i] - 1, -1):  # reverse!
            dp[w] = max(dp[w], dp[w - weights[i]] + values[i])
    return dp[capacity]
```

### State Machine (Stock Problem)
```python
def max_profit_with_cooldown(prices):
    hold = -prices[0]  # holding stock
    sold = 0           # just sold
    rest = 0           # cooldown
    
    for price in prices[1:]:
        prev_hold = hold
        hold = max(hold, rest - price)      # buy or keep holding
        rest = max(rest, sold)              # cooldown or stay resting
        sold = prev_hold + price            # sell
    
    return max(sold, rest)
```

### Interval DP
```python
def burst_balloons(nums):
    nums = [1] + nums + [1]
    n = len(nums)
    dp = [[0] * n for _ in range(n)]
    
    for length in range(2, n):  # window size
        for left in range(0, n - length):
            right = left + length
            for k in range(left + 1, right):  # last to burst
                dp[left][right] = max(
                    dp[left][right],
                    dp[left][k] + dp[k][right] + nums[left] * nums[k] * nums[right]
                )
    
    return dp[0][n-1]
```
