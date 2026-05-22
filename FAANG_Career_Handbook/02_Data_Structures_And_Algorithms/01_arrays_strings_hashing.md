# Arrays, Strings & Hashing — The Foundation of Every Interview

## Why This is #1

80%+ of coding interview problems involve arrays, strings, or hash maps in some way. Master these, and you've covered the majority of what you'll face.

---

## Arrays — Core Concepts

### What an Array Really Is
Contiguous block of memory. Index → direct address calculation → O(1) access.

### Operations Complexity
| Operation | Array | Dynamic Array (Python list) |
|-----------|-------|---------------------------|
| Access by index | O(1) | O(1) |
| Search (unsorted) | O(n) | O(n) |
| Search (sorted) | O(log n) | O(log n) |
| Insert at end | N/A | Amortized O(1) |
| Insert at position | O(n) | O(n) |
| Delete | O(n) | O(n) |

### Python-Specific Array Knowledge
```python
# List (dynamic array)
arr = [1, 2, 3, 4, 5]

# Slicing — O(k) where k is slice size
sub = arr[1:4]  # [2, 3, 4]

# List comprehension — preferred in interviews
squares = [x**2 for x in arr if x > 2]

# Sorting
arr.sort()                    # In-place, O(n log n)
sorted_arr = sorted(arr)      # Returns new list

# Common tricks
arr[::-1]                     # Reverse
arr[-1]                       # Last element
len(arr)                      # Length in O(1)
```

---

## Hashing — The Most Powerful Interview Tool

### Hash Map (Dictionary) — When and Why

A hash map gives you O(1) average lookup/insert/delete. When you see "find if X exists" or "count occurrences" → think hash map.

```python
# Counter pattern — extremely common
from collections import Counter, defaultdict

# Count frequency
freq = Counter("aabbbcccc")  # {'c': 4, 'b': 3, 'a': 2}

# Group items
groups = defaultdict(list)
for word in words:
    groups[len(word)].append(word)

# Two Sum — THE classic hash map problem
def two_sum(nums, target):
    seen = {}
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []
```

### Hash Set — When You Only Need Existence
```python
# O(1) lookup for "have I seen this?"
seen = set()
for num in nums:
    if num in seen:
        return True  # Duplicate found
    seen.add(num)
```

---

## Essential Array Patterns

### Pattern 1: Two Pointers (Sorted Array)
**When:** Array is sorted, need to find pairs/triplets meeting a condition.

```python
# Two Sum on sorted array
def two_sum_sorted(arr, target):
    left, right = 0, len(arr) - 1
    while left < right:
        curr_sum = arr[left] + arr[right]
        if curr_sum == target:
            return [left, right]
        elif curr_sum < target:
            left += 1
        else:
            right -= 1
    return []

# Three Sum (asked at Google, Amazon, Meta)
def three_sum(nums):
    nums.sort()
    result = []
    for i in range(len(nums) - 2):
        if i > 0 and nums[i] == nums[i-1]:
            continue  # Skip duplicates
        left, right = i + 1, len(nums) - 1
        while left < right:
            total = nums[i] + nums[left] + nums[right]
            if total == 0:
                result.append([nums[i], nums[left], nums[right]])
                while left < right and nums[left] == nums[left+1]:
                    left += 1
                while left < right and nums[right] == nums[right-1]:
                    right -= 1
                left += 1
                right -= 1
            elif total < 0:
                left += 1
            else:
                right -= 1
    return result
```

### Pattern 2: Sliding Window
**When:** Find subarray/substring with some property (max sum, contains all chars, etc.)

```python
# Maximum sum subarray of size k
def max_sum_subarray(arr, k):
    window_sum = sum(arr[:k])
    max_sum = window_sum
    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i - k]  # Slide: add right, remove left
        max_sum = max(max_sum, window_sum)
    return max_sum

# Longest substring without repeating characters (LeetCode #3 — asked everywhere)
def length_of_longest_substring(s):
    char_index = {}
    max_len = 0
    left = 0
    for right, char in enumerate(s):
        if char in char_index and char_index[char] >= left:
            left = char_index[char] + 1
        char_index[char] = right
        max_len = max(max_len, right - left + 1)
    return max_len

# Minimum window substring (hard but common)
def min_window(s, t):
    from collections import Counter
    need = Counter(t)
    missing = len(t)
    left = 0
    start, end = 0, float('inf')
    
    for right, char in enumerate(s):
        if need[char] > 0:
            missing -= 1
        need[char] -= 1
        
        while missing == 0:  # Valid window — try to shrink
            if right - left < end - start:
                start, end = left, right
            need[s[left]] += 1
            if need[s[left]] > 0:
                missing += 1
            left += 1
    
    return s[start:end+1] if end != float('inf') else ""
```

### Pattern 3: Prefix Sum
**When:** Multiple queries for range sums.

```python
# Build prefix sum array
def build_prefix(arr):
    prefix = [0] * (len(arr) + 1)
    for i in range(len(arr)):
        prefix[i + 1] = prefix[i] + arr[i]
    return prefix

# Sum of range [l, r] in O(1)
def range_sum(prefix, l, r):
    return prefix[r + 1] - prefix[l]

# Subarray sum equals K (asked at Facebook/Meta)
def subarray_sum(nums, k):
    count = 0
    curr_sum = 0
    prefix_counts = {0: 1}  # sum -> frequency
    
    for num in nums:
        curr_sum += num
        if curr_sum - k in prefix_counts:
            count += prefix_counts[curr_sum - k]
        prefix_counts[curr_sum] = prefix_counts.get(curr_sum, 0) + 1
    
    return count
```

### Pattern 4: Kadane's Algorithm (Maximum Subarray)
```python
def max_subarray(nums):
    max_sum = curr_sum = nums[0]
    for num in nums[1:]:
        curr_sum = max(num, curr_sum + num)  # Extend or start fresh
        max_sum = max(max_sum, curr_sum)
    return max_sum
```

---

## Essential String Patterns

### Pattern 5: Anagram/Character Frequency
```python
# Check if two strings are anagrams
def is_anagram(s, t):
    return Counter(s) == Counter(t)

# Group anagrams (asked at Amazon, Google)
def group_anagrams(strs):
    groups = defaultdict(list)
    for s in strs:
        key = tuple(sorted(s))  # Or use character count tuple
        groups[key].append(s)
    return list(groups.values())

# Valid palindrome
def is_palindrome(s):
    s = ''.join(c.lower() for c in s if c.isalnum())
    return s == s[::-1]
```

### Pattern 6: String Building
```python
# NEVER concatenate strings in a loop (O(n²))
# Use list + join (O(n))
parts = []
for char in data:
    parts.append(transform(char))
result = ''.join(parts)
```

---

## Top Interview Problems (Must Solve)

| # | Problem | Pattern | Company |
|---|---------|---------|---------|
| 1 | Two Sum | Hash Map | Everyone |
| 2 | Best Time to Buy/Sell Stock | Kadane's variant | Amazon, Goldman |
| 3 | Contains Duplicate | Hash Set | Easy warm-up |
| 4 | Product of Array Except Self | Prefix/Suffix | Amazon, Microsoft |
| 5 | Maximum Subarray | Kadane's | Everyone |
| 6 | Three Sum | Two Pointers | Google, Meta |
| 7 | Container With Most Water | Two Pointers | Amazon |
| 8 | Longest Substring Without Repeat | Sliding Window | Everyone |
| 9 | Minimum Window Substring | Sliding Window | Meta, Google |
| 10 | Group Anagrams | Hash Map | Amazon, Meta |
| 11 | Valid Parentheses | Stack (next section) | Everyone |
| 12 | Subarray Sum Equals K | Prefix Sum + Hash | Meta |
| 13 | Trapping Rain Water | Two Pointers/Stack | Google, Amazon |
| 14 | Merge Intervals | Sort + Sweep | Everyone |
| 15 | Next Permutation | Array manipulation | Google |

---

## Mistakes to Avoid

1. **Forgetting to handle empty input** — Always check `if not arr: return`
2. **Off-by-one in sliding window** — Window size = right - left + 1
3. **Modifying array while iterating** — Use a copy or iterate backwards
4. **Using `list.index()` in a loop** — That's O(n) per call, use a hash map
5. **String concatenation in loops** — Use `''.join()` instead
6. **Not considering negative numbers** — Especially in max/min subarray problems
7. **Assuming sorted input** — Always ask "Is the input sorted?"

---

## How These Show Up at Real Companies

**Google:** Loves sliding window variations, string manipulation, Two Sum variants with constraints
**Amazon:** Array manipulation, merge intervals, product of array except self
**Meta/Facebook:** Subarray sum problems, string matching, anagram variants
**Microsoft:** Two pointers, binary search on arrays, matrix problems
**Apple:** String parsing, array transformation, in-place modifications
