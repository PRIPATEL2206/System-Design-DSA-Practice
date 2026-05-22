# Tries & String Algorithms — FAANG Questions

## Core Concepts

### Trie (Prefix Tree)
Use for **prefix-based search**, autocomplete, and word dictionary problems.

### String Patterns
```
Pattern                    Technique
──────────────────────────────────────────────
Prefix/suffix matching    → Trie
Pattern matching          → KMP or Rabin-Karp
Anagram detection         → Frequency array / sorting
Palindrome               → Two pointers or DP
Subsequence              → Two pointers or DP
String manipulation      → Stack or two pointer
```

---

## Questions

### 🟢 Easy

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 1 | Valid Palindrome | M, A, G | Two pointers, skip non-alnum |
| 2 | Valid Anagram | A, M, G | Frequency count |
| 3 | Longest Common Prefix | A, M, G | Vertical scan or sort |
| 4 | Roman to Integer | A, M, G | Map + subtraction rule |
| 5 | Reverse Words in a String | A, M, G | Split + reverse + join |
| 6 | Is Subsequence | A, G, M | Two pointer |

---

### 🟡 Medium

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 7 | Implement Trie (Prefix Tree) | A, M, G | Node with children dict + end flag |
| 8 | Design Add and Search Words | A, M, G | Trie + DFS for '.' wildcard |
| 9 | Word Search II | A, G, M | Trie + grid DFS |
| 10 | Longest Palindromic Substring | A, M, G | Expand around center |
| 11 | Palindromic Substrings (count) | A, M, G | Expand around each center |
| 12 | Group Anagrams | M, A, G | Sorted key or frequency key |
| 13 | String to Integer (atoi) | A, M, G, MS | State machine: whitespace, sign, digits |
| 14 | Longest Repeating Character Replacement | G, A, M | Sliding window + max freq |
| 15 | Minimum Remove to Make Valid Parentheses | M, A, G | Stack for indices to remove |
| 16 | Decode String | A, G, M | Stack of (string, count) |
| 17 | Generate Parentheses | A, M, G | Backtracking |
| 18 | Count and Say | A, M, G | Run-length encoding |
| 19 | Zigzag Conversion | A, M | Simulate rows or math |
| 20 | Multiply Strings | A, M, G | Grade-school multiplication |
| 21 | Replace Words | A, M | Trie: find shortest prefix |
| 22 | Map Sum Pairs | A, G | Trie with values |
| 23 | Top K Frequent Words | A, M, G | Heap or bucket sort |
| 24 | Longest Word in Dictionary | A, G | Trie: BFS for longest buildable |
| 25 | Search Suggestions System | A, G | Trie + DFS for suggestions or BS |

---

### 🔴 Hard

| # | Problem | Companies | Key Idea |
|---|---------|-----------|----------|
| 26 | Word Search II (multiple words) | A, G, M | Trie + grid backtracking |
| 27 | Palindrome Pairs | A, G | Trie of reversed words |
| 28 | Stream of Characters | G | Reverse trie + check suffix |
| 29 | Concatenated Words | A, G | Trie + DP word break |
| 30 | Shortest Palindrome (KMP) | G, A | KMP failure function on s + "#" + rev(s) |

---

## Templates

### Trie Implementation
```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()
    
    def insert(self, word):
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        node.is_end = True
    
    def search(self, word):
        node = self._find(word)
        return node is not None and node.is_end
    
    def starts_with(self, prefix):
        return self._find(prefix) is not None
    
    def _find(self, prefix):
        node = self.root
        for char in prefix:
            if char not in node.children:
                return None
            node = node.children[char]
        return node
```

### Expand Around Center (Palindrome)
```python
def longest_palindrome(s):
    def expand(l, r):
        while l >= 0 and r < len(s) and s[l] == s[r]:
            l -= 1
            r += 1
        return s[l+1:r]
    
    result = ""
    for i in range(len(s)):
        # Odd length
        odd = expand(i, i)
        if len(odd) > len(result):
            result = odd
        # Even length
        even = expand(i, i + 1)
        if len(even) > len(result):
            result = even
    return result
```

### KMP Pattern Matching
```python
def kmp_search(text, pattern):
    # Build failure function
    lps = [0] * len(pattern)
    j = 0
    for i in range(1, len(pattern)):
        while j > 0 and pattern[i] != pattern[j]:
            j = lps[j-1]
        if pattern[i] == pattern[j]:
            j += 1
        lps[i] = j
    
    # Search
    j = 0
    matches = []
    for i in range(len(text)):
        while j > 0 and text[i] != pattern[j]:
            j = lps[j-1]
        if text[i] == pattern[j]:
            j += 1
        if j == len(pattern):
            matches.append(i - j + 1)
            j = lps[j-1]
    
    return matches
```
