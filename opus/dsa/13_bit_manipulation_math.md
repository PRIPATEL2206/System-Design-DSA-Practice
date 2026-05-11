# Bit Manipulation & Math — FAANG Questions

## Core Concepts

### Bit Manipulation
```
Operation          Python    What it does
────────────────────────────────────────────────────
AND                a & b     Both bits 1 → 1
OR                 a | b     Either bit 1 → 1
XOR                a ^ b     Different bits → 1 (toggle)
NOT                ~a        Flip all bits
Left Shift         a << n    Multiply by 2^n
Right Shift        a >> n    Divide by 2^n

Key Tricks:
───────────
n & (n-1)          Remove lowest set bit
n & (-n)           Isolate lowest set bit
n ^ n = 0          XOR with itself = 0
n ^ 0 = n          XOR with 0 = itself
x & 1              Check if odd
```

### Math Patterns
```
GCD                → Euclidean algorithm
Prime check        → Trial division up to √n
Modular arithmetic → (a*b) % m = ((a%m) * (b%m)) % m
Power              → Fast exponentiation (square and multiply)
Combinatorics      → nCr with Pascal's triangle or formula
```

---

## Questions

### 🟢 Easy

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 1 | Single Number | A, M, G | XOR all → duplicate cancels |
| 2 | Number of 1 Bits | A, M, G | n & (n-1) removes lowest bit |
| 3 | Counting Bits | A, M | dp[i] = dp[i >> 1] + (i & 1) |
| 4 | Reverse Bits | A, G | Bit by bit extraction |
| 5 | Missing Number | A, M, G | XOR [0..n] with array |
| 6 | Power of Two | A, G | n & (n-1) == 0 |
| 7 | Add Binary | A, M, G | Simulate with carry |
| 8 | Fizz Buzz | A, M | Modulo check |
| 9 | Happy Number | A, G | Detect cycle (Floyd's) |
| 10 | Excel Sheet Column Number | A, M | Base-26 conversion |

---

### 🟡 Medium

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 11 | Single Number II (appears 3 times) | A, G | Count bits mod 3 |
| 12 | Single Number III (two singles) | A, G | XOR all → split by a set bit |
| 13 | Subsets (via bitmask) | A, M, G | Iterate 0 to 2^n - 1 |
| 14 | Bitwise AND of Numbers Range | A, M, G | Common prefix of L and R |
| 15 | Pow(x, n) | A, M, G, MS | Fast exponentiation |
| 16 | Divide Two Integers (no * / %) | A, M, G | Bit shifting |
| 17 | Sum of Two Integers (no + -) | A, M, G | XOR for sum, AND<<1 for carry |
| 18 | UTF-8 Validation | G, A | Bit masking for byte patterns |
| 19 | Maximum XOR of Two Numbers | G, A | Trie of bits |
| 20 | Total Hamming Distance | A, G | Count 1s at each bit position |
| 21 | Integer Break | A, G | Math: maximize product (use 3s) |
| 22 | Count Primes | A, M, G | Sieve of Eratosthenes |
| 23 | Factorial Trailing Zeroes | A, M, G | Count factors of 5 |
| 24 | Ugly Number II | A, G | Three pointers DP |
| 25 | Fraction to Recurring Decimal | A, M, G | Long division + detect cycle |

---

### 🔴 Hard

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 26 | Maximum XOR With Element From Array | G | Offline + Trie |
| 27 | Shortest Path Visiting All Nodes | G | BFS + bitmask |
| 28 | Number of Digit One | G | Digit DP |
| 29 | Stickers to Spell Word | G | Bitmask DP |
| 30 | Minimum Number of K Consecutive Bit Flips | G | Greedy + queue |

---

## Templates

### Fast Power
```python
def power(base, exp, mod=None):
    result = 1
    while exp > 0:
        if exp & 1:
            result = result * base if not mod else (result * base) % mod
        base = base * base if not mod else (base * base) % mod
        exp >>= 1
    return result
```

### Count Set Bits (Brian Kernighan's)
```python
def count_bits(n):
    count = 0
    while n:
        n &= (n - 1)  # remove lowest set bit
        count += 1
    return count
```

### Sieve of Eratosthenes
```python
def count_primes(n):
    if n < 2:
        return 0
    is_prime = [True] * n
    is_prime[0] = is_prime[1] = False
    for i in range(2, int(n**0.5) + 1):
        if is_prime[i]:
            for j in range(i*i, n, i):
                is_prime[j] = False
    return sum(is_prime)
```

### GCD / LCM
```python
def gcd(a, b):
    while b:
        a, b = b, a % b
    return a

def lcm(a, b):
    return a * b // gcd(a, b)
```

### Generate Subsets via Bitmask
```python
def subsets_bitmask(nums):
    n = len(nums)
    result = []
    for mask in range(1 << n):
        subset = []
        for i in range(n):
            if mask & (1 << i):
                subset.append(nums[i])
        result.append(subset)
    return result
```
