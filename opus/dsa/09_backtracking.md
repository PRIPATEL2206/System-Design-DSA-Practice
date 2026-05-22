# Backtracking — FAANG Questions

## Core Concepts

Backtracking = **DFS + Pruning**. Explore all possibilities, but abandon paths that can't lead to a valid solution.

```
Template:
─────────
def backtrack(path, choices):
    if is_solution(path):
        result.append(path[:])  # copy!
        return
    
    for choice in choices:
        if is_valid(choice):
            path.append(choice)       # make choice
            backtrack(path, ...)      # explore
            path.pop()                # undo choice (backtrack)

Key Patterns:
─────────────
Subsets       → Include or exclude each element
Permutations  → Place each unused element
Combinations  → Choose k from n (with index tracking)
Partitioning  → Split string at every valid position
Grid/Board    → Try each direction, mark visited
Constraint    → Sudoku, N-Queens (check validity before placing)
```

**Tip:** To avoid duplicates, sort the input and skip `if i > start and nums[i] == nums[i-1]`

---

## Questions

### 🟢 Easy

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 1 | Letter Case Permutation | A, G | Branch: lowercase or uppercase |
| 2 | Binary Watch | A, G | Count set bits |

---

### 🟡 Medium

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 3 | Subsets | A, M, G | Include/exclude each element |
| 4 | Subsets II (with duplicates) | A, M, G | Sort + skip duplicates |
| 5 | Permutations | A, M, G | Use visited set or swap |
| 6 | Permutations II (with duplicates) | A, M, G | Sort + skip same at same level |
| 7 | Combinations | A, M, G | Choose k from [1..n] |
| 8 | Combination Sum (unlimited use) | A, M, G | Start from same index (not i+1) |
| 9 | Combination Sum II (each once) | A, M, G | Start from i+1, skip duplicates |
| 10 | Combination Sum III | A, G | Choose k numbers summing to n |
| 11 | Generate Parentheses | A, M, G | open < n: add '('; close < open: add ')' |
| 12 | Palindrome Partitioning | A, M, G | Try every split, check palindrome |
| 13 | Letter Combinations of Phone Number | A, M, G | Map digits to letters, combine |
| 14 | Word Search | A, M, G, MS | DFS grid, mark visited |
| 15 | Restore IP Addresses | A, M, G | 4 segments, each 0-255 |
| 16 | Path Sum II | A, M, G | Track path, check at leaves |
| 17 | Beautiful Arrangement | A, G | Permutation with divisibility check |
| 18 | Partition to K Equal Sum Subsets | A, G | Assign each number to a bucket |
| 19 | Maximum Length of Concatenated String | A, G | Subset selection with no char overlap |
| 20 | Split Array into Fibonacci Sequence | A, G | Try first two numbers, validate rest |

---

### 🔴 Hard

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 21 | N-Queens | A, G, M | Place queen per row, check col/diag |
| 22 | N-Queens II (count only) | A, G | Same but just count |
| 23 | Sudoku Solver | A, G, M | Fill empty cells, validate row/col/box |
| 24 | Word Search II (multiple words) | A, G, M | Trie + DFS on grid |
| 25 | Expression Add Operators | G, A, M | Insert +,-,* between digits |
| 26 | Stickers to Spell Word | G | Bitmask DP or backtracking with memo |
| 27 | Word Break II | A, M, G | Backtrack with memoization |
| 28 | Unique Paths III (visit all cells) | G, A | DFS counting empty cells |

---

## Templates

### Subsets
```python
def subsets(nums):
    result = []
    def backtrack(start, path):
        result.append(path[:])
        for i in range(start, len(nums)):
            path.append(nums[i])
            backtrack(i + 1, path)
            path.pop()
    backtrack(0, [])
    return result
```

### Permutations
```python
def permute(nums):
    result = []
    def backtrack(path, used):
        if len(path) == len(nums):
            result.append(path[:])
            return
        for i in range(len(nums)):
            if used[i]:
                continue
            used[i] = True
            path.append(nums[i])
            backtrack(path, used)
            path.pop()
            used[i] = False
    backtrack([], [False] * len(nums))
    return result
```

### Combination Sum
```python
def combination_sum(candidates, target):
    result = []
    def backtrack(start, path, remaining):
        if remaining == 0:
            result.append(path[:])
            return
        for i in range(start, len(candidates)):
            if candidates[i] > remaining:
                break  # pruning (requires sorted input)
            path.append(candidates[i])
            backtrack(i, path, remaining - candidates[i])  # same i = reuse allowed
            path.pop()
    candidates.sort()
    backtrack(0, [], target)
    return result
```

### N-Queens
```python
def solve_n_queens(n):
    result = []
    cols = set()
    diag1 = set()  # row - col
    diag2 = set()  # row + col
    
    def backtrack(row, queens):
        if row == n:
            result.append(queens[:])
            return
        for col in range(n):
            if col in cols or (row-col) in diag1 or (row+col) in diag2:
                continue
            cols.add(col)
            diag1.add(row - col)
            diag2.add(row + col)
            queens.append(col)
            backtrack(row + 1, queens)
            queens.pop()
            cols.remove(col)
            diag1.remove(row - col)
            diag2.remove(row + col)
    
    backtrack(0, [])
    return result
```

### Word Search on Grid
```python
def exist(board, word):
    rows, cols = len(board), len(board[0])
    
    def dfs(r, c, i):
        if i == len(word):
            return True
        if r < 0 or r >= rows or c < 0 or c >= cols:
            return False
        if board[r][c] != word[i]:
            return False
        
        board[r][c] = '#'  # mark visited
        found = any(dfs(r+dr, c+dc, i+1) for dr, dc in [(0,1),(0,-1),(1,0),(-1,0)])
        board[r][c] = word[i]  # restore
        return found
    
    return any(dfs(r, c, 0) for r in range(rows) for c in range(cols))
```
