# Coding Practice Problems — Python & DSA

> Solve these WITHOUT looking at solutions. Time yourself. Then compare with the solution below.

---

## Difficulty Levels
- **Easy** = Warm-up, should take < 10 min
- **Medium** = Expected in interviews, 15-25 min
- **Hard** = Differentiator, 25-40 min

---

## Problem 1: Two Sum with Streaming Data (Easy → Medium)

> You're processing a stream of transaction amounts. Find if any two transactions sum to a target amount.
> Follow-up: What if the stream is infinite and you only have limited memory?

**Constraints:**
- Stream of integers (can be very large)
- Return True/False + the pair
- Memory efficient for follow-up

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
# Basic approach - Hash Set O(n) time, O(n) space
def two_sum_stream(stream, target):
    seen = set()
    for num in stream:
        complement = target - num
        if complement in seen:
            return True, (complement, num)
        seen.add(num)
    return False, None

# Memory-bounded approach - Bloom Filter (probabilistic)
class BloomFilter:
    def __init__(self, size=1000000):
        self.size = size
        self.bit_array = [0] * size
    
    def _hashes(self, item):
        h1 = hash(item) % self.size
        h2 = hash(str(item) + "salt") % self.size
        return [h1, h2]
    
    def add(self, item):
        for h in self._hashes(item):
            self.bit_array[h] = 1
    
    def might_contain(self, item):
        return all(self.bit_array[h] for h in self._hashes(item))

def two_sum_memory_bounded(stream, target):
    bf = BloomFilter()
    for num in stream:
        complement = target - num
        if bf.might_contain(complement):
            return True, (complement, num)  # May have false positives
        bf.add(num)
    return False, None

# Sliding window approach (if looking within last N elements)
from collections import deque

def two_sum_sliding_window(stream, target, window_size=10000):
    window = deque()
    window_set = set()
    
    for num in stream:
        complement = target - num
        if complement in window_set:
            return True, (complement, num)
        
        window.append(num)
        window_set.add(num)
        
        if len(window) > window_size:
            removed = window.popleft()
            window_set.discard(removed)
    
    return False, None
```

**Complexity:** O(n) time, O(n) or O(window) space  
**Interview tip:** Discuss trade-offs between exact vs probabilistic (Bloom Filter) for infinite streams.
</details>

---

## Problem 2: Merge K Sorted Iterators (Medium)

> You have K sorted data streams (like sorted files from different ETL sources). Merge them into one sorted output without loading all data into memory.

**Real-world context:** Merging sorted partitions from multiple Glue jobs.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
import heapq
from typing import List, Iterator, Any

def merge_k_sorted(iterators: List[Iterator]) -> Iterator:
    """Merge K sorted iterators into single sorted output."""
    heap = []
    
    # Initialize heap with first element from each iterator
    for i, it in enumerate(iterators):
        try:
            val = next(it)
            heapq.heappush(heap, (val, i))
        except StopIteration:
            pass
    
    while heap:
        val, idx = heapq.heappop(heap)
        yield val
        
        try:
            next_val = next(iterators[idx])
            heapq.heappush(heap, (next_val, idx))
        except StopIteration:
            pass

# Usage - simulating sorted files
def sorted_file_reader(filepath):
    """Generator that reads sorted file line by line."""
    with open(filepath) as f:
        for line in f:
            yield int(line.strip())

# Merge sorted partitions
# files = ["partition_0.csv", "partition_1.csv", "partition_2.csv"]
# readers = [sorted_file_reader(f) for f in files]
# for record in merge_k_sorted(readers):
#     write_to_output(record)

# With custom objects (like HCP records)
from dataclasses import dataclass, field

@dataclass(order=True)
class Record:
    sort_key: str
    data: dict = field(compare=False)

def merge_k_sorted_records(sources: List[Iterator[Record]]) -> Iterator[Record]:
    heap = []
    for i, source in enumerate(sources):
        try:
            record = next(source)
            heapq.heappush(heap, (record, i))
        except StopIteration:
            pass
    
    while heap:
        record, idx = heapq.heappop(heap)
        yield record
        try:
            next_record = next(sources[idx])
            heapq.heappush(heap, (next_record, idx))
        except StopIteration:
            pass
```

**Complexity:** O(N log K) where N = total elements, K = number of sources  
**Interview tip:** Emphasize memory efficiency — only K elements in heap at any time. Perfect for processing data larger than RAM.
</details>

---

## Problem 3: Implement a Thread-Safe Producer-Consumer Queue (Medium)

> Design a bounded queue where multiple producers add data and multiple consumers process it. This is the foundation of your ETL pipeline's buffering mechanism.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
import threading
from collections import deque
from typing import Any, Optional
import time

class BoundedQueue:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.queue = deque()
        self.lock = threading.Lock()
        self.not_full = threading.Condition(self.lock)
        self.not_empty = threading.Condition(self.lock)
        self.closed = False

    def put(self, item: Any, timeout: float = None) -> bool:
        with self.not_full:
            if self.closed:
                raise ValueError("Queue is closed")
            
            while len(self.queue) >= self.capacity:
                if not self.not_full.wait(timeout=timeout):
                    return False  # Timeout
            
            self.queue.append(item)
            self.not_empty.notify()
            return True

    def get(self, timeout: float = None) -> Optional[Any]:
        with self.not_empty:
            while len(self.queue) == 0:
                if self.closed:
                    return None
                if not self.not_empty.wait(timeout=timeout):
                    return None  # Timeout
            
            item = self.queue.popleft()
            self.not_full.notify()
            return item

    def close(self):
        with self.lock:
            self.closed = True
            self.not_empty.notify_all()
            self.not_full.notify_all()

    @property
    def size(self):
        with self.lock:
            return len(self.queue)

# Usage: ETL Pipeline buffering
def producer(queue, source_name):
    """Simulates reading records from a source."""
    for i in range(100):
        record = {"id": f"{source_name}_{i}", "data": f"record_{i}"}
        queue.put(record)
        print(f"[{source_name}] Produced record {i}")
    print(f"[{source_name}] Done producing")

def consumer(queue, consumer_name):
    """Simulates processing records."""
    processed = 0
    while True:
        record = queue.get(timeout=5)
        if record is None:
            break
        # Process record
        processed += 1
        print(f"[{consumer_name}] Processed: {record['id']}")
    print(f"[{consumer_name}] Done. Processed {processed} records")

# Run
queue = BoundedQueue(capacity=50)
producers = [
    threading.Thread(target=producer, args=(queue, f"source_{i}"))
    for i in range(3)
]
consumers = [
    threading.Thread(target=consumer, args=(queue, f"worker_{i}"))
    for i in range(2)
]

for t in producers + consumers:
    t.start()
for t in producers:
    t.join()
queue.close()
for t in consumers:
    t.join()
```

**Complexity:** O(1) per put/get operation  
**Interview tip:** Discuss why Condition variables are needed (vs just a lock). Mention backpressure — producers block when queue is full.
</details>

---

## Problem 4: Implement a Trie with Autocomplete (Medium)

> Build an autocomplete system. Given a prefix, return top-K suggestions by frequency.

**Real-world context:** Search autocomplete for your internal portal.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
import heapq
from typing import List, Tuple
from collections import defaultdict

class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_word = False
        self.frequency = 0

class AutocompleteTrie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str, frequency: int = 1):
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        node.is_word = True
        node.frequency += frequency

    def search(self, word: str) -> bool:
        node = self._find_node(word)
        return node is not None and node.is_word

    def _find_node(self, prefix: str) -> TrieNode:
        node = self.root
        for char in prefix:
            if char not in node.children:
                return None
            node = node.children[char]
        return node

    def autocomplete(self, prefix: str, top_k: int = 5) -> List[Tuple[str, int]]:
        node = self._find_node(prefix)
        if not node:
            return []
        
        # DFS to find all words with this prefix
        results = []
        self._dfs(node, prefix, results)
        
        # Return top-K by frequency
        return heapq.nlargest(top_k, results, key=lambda x: x[1])

    def _dfs(self, node: TrieNode, current_word: str, results: list):
        if node.is_word:
            results.append((current_word, node.frequency))
        for char, child in node.children.items():
            self._dfs(child, current_word + char, results)

    def delete(self, word: str) -> bool:
        def _delete(node, word, depth):
            if depth == len(word):
                if not node.is_word:
                    return False
                node.is_word = False
                return len(node.children) == 0
            
            char = word[depth]
            if char not in node.children:
                return False
            
            should_delete = _delete(node.children[char], word, depth + 1)
            if should_delete:
                del node.children[char]
                return not node.is_word and len(node.children) == 0
            return False
        
        return _delete(self.root, word, 0)

# Usage
trie = AutocompleteTrie()
search_terms = [
    ("python data engineering", 100),
    ("python pyspark tutorial", 80),
    ("python aws lambda", 75),
    ("python fastapi", 60),
    ("python decorator", 50),
    ("pyspark optimization", 45),
    ("pyspark join types", 40),
]
for term, freq in search_terms:
    trie.insert(term, freq)

print(trie.autocomplete("py", 3))
# [('python data engineering', 100), ('python pyspark tutorial', 80), ('python aws lambda', 75)]

print(trie.autocomplete("pyspark", 2))
# [('pyspark optimization', 45), ('pyspark join types', 40)]
```

**Complexity:** Insert/Search O(L), Autocomplete O(N) where N = words under prefix  
**Interview tip:** For top-K optimization, mention storing top results at each node (pre-computed) for O(1) autocomplete at the cost of more memory.
</details>

---

## Problem 5: Find Duplicate Records in a Data Stream (Medium)

> You're deduplicating HCP records in your pipeline. Detect duplicates based on composite key (name + dob + region) from a stream that doesn't fit in memory.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
import hashlib
from typing import Iterator, Dict, Tuple, Set
import mmh3  # MurmurHash for Bloom Filter

# Approach 1: Hash-based deduplication (fits in memory)
def deduplicate_exact(records: Iterator[Dict]) -> Iterator[Dict]:
    """Exact deduplication using hash set. O(n) memory."""
    seen = set()
    for record in records:
        key = f"{record['name']}|{record['dob']}|{record['region']}"
        key_hash = hashlib.md5(key.encode()).hexdigest()
        
        if key_hash not in seen:
            seen.add(key_hash)
            yield record

# Approach 2: Bloom Filter (probabilistic, bounded memory)
class BloomFilter:
    def __init__(self, expected_items: int, fp_rate: float = 0.01):
        self.size = self._optimal_size(expected_items, fp_rate)
        self.num_hashes = self._optimal_hashes(self.size, expected_items)
        self.bit_array = bytearray(self.size // 8 + 1)
    
    def _optimal_size(self, n, p):
        import math
        return int(-n * math.log(p) / (math.log(2) ** 2))
    
    def _optimal_hashes(self, m, n):
        import math
        return int((m / n) * math.log(2))
    
    def _get_bit(self, index):
        return (self.bit_array[index // 8] >> (index % 8)) & 1
    
    def _set_bit(self, index):
        self.bit_array[index // 8] |= (1 << (index % 8))
    
    def add(self, item: str):
        for i in range(self.num_hashes):
            idx = mmh3.hash(item, i) % self.size
            self._set_bit(idx)
    
    def might_contain(self, item: str) -> bool:
        for i in range(self.num_hashes):
            idx = mmh3.hash(item, i) % self.size
            if not self._get_bit(idx):
                return False
        return True

def deduplicate_probabilistic(records: Iterator[Dict], expected_count: int) -> Iterator[Dict]:
    """Probabilistic dedup. ~1% false positive (skips non-dupes), 0% false negative."""
    bloom = BloomFilter(expected_count, fp_rate=0.001)
    
    for record in records:
        key = f"{record['name']}|{record['dob']}|{record['region']}"
        
        if not bloom.might_contain(key):
            bloom.add(key)
            yield record
        # else: might be duplicate (or false positive) — skip

# Approach 3: External sort + merge dedup (for truly massive data)
def deduplicate_external_sort(input_file: str, output_file: str, key_columns: list):
    """
    For data too large for memory:
    1. Sort file by composite key (external merge sort)
    2. Linear scan: consecutive duplicates are adjacent
    """
    import subprocess
    
    # Unix sort (external merge sort, handles files larger than RAM)
    key_indices = ",".join(str(k) for k in key_columns)
    subprocess.run(f"sort -t',' -k{key_indices} -u {input_file} > {output_file}", shell=True)

# Approach 4: Window-based dedup (for near-real-time streams)
from collections import OrderedDict

class WindowDeduplicator:
    """Dedup within a time window. Records outside window are assumed unique."""
    def __init__(self, window_size: int = 100000):
        self.window_size = window_size
        self.seen = OrderedDict()
    
    def is_duplicate(self, record: Dict) -> bool:
        key = f"{record['name']}|{record['dob']}|{record['region']}"
        
        if key in self.seen:
            return True
        
        self.seen[key] = True
        if len(self.seen) > self.window_size:
            self.seen.popitem(last=False)
        
        return False
```

**Interview tip:** Start with exact (hash set), then discuss memory constraints → Bloom Filter. Mention false positive trade-off. For batch ETL, external sort + dedup is the classic approach.
</details>

---

## Problem 6: Implement a Consistent Hash Ring (Hard)

> Design a load balancer that distributes requests across servers. When a server is added/removed, minimize data movement.

**Real-world context:** Distributing WebSocket connections across chat servers, or partitioning data across Redshift nodes.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
import hashlib
from bisect import bisect_right
from typing import Optional, List

class ConsistentHashRing:
    def __init__(self, virtual_nodes: int = 150):
        self.virtual_nodes = virtual_nodes
        self.ring = []  # sorted list of (hash, node_id)
        self.hash_to_node = {}
        self.nodes = set()

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node_id: str):
        self.nodes.add(node_id)
        for i in range(self.virtual_nodes):
            virtual_key = f"{node_id}#vn{i}"
            h = self._hash(virtual_key)
            self.ring.append(h)
            self.hash_to_node[h] = node_id
        self.ring.sort()

    def remove_node(self, node_id: str):
        self.nodes.discard(node_id)
        self.ring = [h for h in self.ring if self.hash_to_node.get(h) != node_id]
        self.hash_to_node = {h: n for h, n in self.hash_to_node.items() if n != node_id}

    def get_node(self, key: str) -> Optional[str]:
        if not self.ring:
            return None
        
        h = self._hash(key)
        idx = bisect_right(self.ring, h)
        
        if idx == len(self.ring):
            idx = 0  # Wrap around
        
        return self.hash_to_node[self.ring[idx]]

    def get_nodes(self, key: str, replicas: int = 3) -> List[str]:
        """Get multiple nodes for replication."""
        if not self.ring:
            return []
        
        h = self._hash(key)
        idx = bisect_right(self.ring, h)
        
        result = []
        seen = set()
        
        for i in range(len(self.ring)):
            pos = (idx + i) % len(self.ring)
            node = self.hash_to_node[self.ring[pos]]
            if node not in seen:
                seen.add(node)
                result.append(node)
            if len(result) == replicas:
                break
        
        return result

# Usage: Distribute WebSocket connections
ring = ConsistentHashRing(virtual_nodes=150)
ring.add_node("server-1")
ring.add_node("server-2")
ring.add_node("server-3")

# Route users to servers
users = ["user_001", "user_002", "user_003", "user_100"]
for user in users:
    server = ring.get_node(user)
    print(f"{user} → {server}")

# Add a new server - only ~1/N keys move
ring.add_node("server-4")
print("\nAfter adding server-4:")
for user in users:
    server = ring.get_node(user)
    print(f"{user} → {server}")

# For data replication:
print(ring.get_nodes("important_data", replicas=3))
# ['server-2', 'server-1', 'server-3'] — store on 3 different servers
```

**Complexity:** O(log N) lookup with binary search, O(K * V) space where K=nodes, V=virtual nodes  
**Interview tip:** Explain WHY virtual nodes (prevents hotspots when nodes are few). Mention real usage in Cassandra, DynamoDB, Memcached.
</details>

---

## Problem 7: Implement a Simple Map-Reduce Framework (Hard)

> Implement a local map-reduce framework. This tests your understanding of how PySpark works under the hood.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
from collections import defaultdict
from typing import Callable, List, Tuple, Any, Iterator
import itertools

class MapReduce:
    def __init__(self, num_workers: int = 4):
        self.num_workers = num_workers

    def map_reduce(self, data: List[Any], mapper: Callable, reducer: Callable,
                   combiner: Callable = None) -> dict:
        """
        mapper: item → List[(key, value)]
        combiner: (key, List[values]) → (key, partial_value) [optional local aggregation]
        reducer: (key, List[values]) → (key, final_value)
        """
        # Phase 1: MAP
        chunks = self._partition_data(data, self.num_workers)
        with ProcessPoolExecutor(max_workers=self.num_workers) as executor:
            map_results = list(executor.map(self._map_chunk, 
                                           [(chunk, mapper) for chunk in chunks]))
        
        # Phase 2: SHUFFLE (group by key)
        shuffled = defaultdict(list)
        for chunk_result in map_results:
            for key, value in chunk_result:
                shuffled[key].append(value)
        
        # Phase 2.5: COMBINE (optional local reduction)
        if combiner:
            shuffled = {k: [combiner(k, v)] for k, v in shuffled.items()}
        
        # Phase 3: REDUCE
        results = {}
        for key, values in shuffled.items():
            results[key] = reducer(key, values)
        
        return results

    def _partition_data(self, data: List, num_partitions: int) -> List[List]:
        chunk_size = len(data) // num_partitions + 1
        return [data[i:i+chunk_size] for i in range(0, len(data), chunk_size)]

    @staticmethod
    def _map_chunk(args):
        chunk, mapper = args
        results = []
        for item in chunk:
            results.extend(mapper(item))
        return results

# Example 1: Word Count
def word_count_mapper(line):
    return [(word.lower(), 1) for word in line.split()]

def word_count_reducer(key, values):
    return sum(values)

mr = MapReduce(num_workers=4)
lines = [
    "hello world hello",
    "world of data engineering",
    "data pipelines are awesome",
    "hello data world"
]
result = mr.map_reduce(lines, word_count_mapper, word_count_reducer)
# {'hello': 3, 'world': 3, 'data': 3, 'of': 1, 'engineering': 1, ...}

# Example 2: ETL - Aggregate HCP records by region
def region_mapper(record):
    return [(record["region"], record["value"])]

def sum_reducer(key, values):
    return sum(values)

def sum_combiner(key, values):
    return sum(values)

records = [
    {"hcp_id": "1", "region": "NORTH", "value": 100},
    {"hcp_id": "2", "region": "SOUTH", "value": 200},
    {"hcp_id": "3", "region": "NORTH", "value": 150},
    {"hcp_id": "4", "region": "SOUTH", "value": 300},
]
result = mr.map_reduce(records, region_mapper, sum_reducer, sum_combiner)
# {'NORTH': 250, 'SOUTH': 500}

# Example 3: Inverted Index (for search)
def index_mapper(doc):
    doc_id, text = doc
    return [(word.lower(), doc_id) for word in set(text.split())]

def list_reducer(key, values):
    return list(set(values))

documents = [
    ("doc1", "python data engineering aws"),
    ("doc2", "python machine learning tensorflow"),
    ("doc3", "aws lambda data pipeline"),
]
index = mr.map_reduce(documents, index_mapper, list_reducer)
# {'python': ['doc1', 'doc2'], 'data': ['doc1', 'doc3'], 'aws': ['doc1', 'doc3'], ...}
```

**Complexity:** O(N/P) per worker for map, O(N) for shuffle, O(unique_keys) for reduce  
**Interview tip:** Explain how this maps to Spark: RDD → map() → shuffle → reduceByKey(). Discuss why combiner helps (reduces network shuffle in distributed setting).
</details>

---

## Problem 8: Design a Retry Mechanism with Exponential Backoff + Jitter (Easy)

> Implement a production-grade retry decorator for your ETL pipeline's API calls.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
import time
import random
import functools
import logging
from typing import Tuple, Type

logger = logging.getLogger(__name__)

def retry(
    max_attempts: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    exponential_base: float = 2.0,
    jitter: bool = True,
    retryable_exceptions: Tuple[Type[Exception], ...] = (Exception,),
    on_retry: callable = None
):
    """Production-grade retry decorator with exponential backoff + jitter."""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except retryable_exceptions as e:
                    last_exception = e
                    
                    if attempt == max_attempts:
                        logger.error(f"[{func.__name__}] All {max_attempts} attempts failed. Last error: {e}")
                        raise
                    
                    # Calculate delay
                    delay = min(base_delay * (exponential_base ** (attempt - 1)), max_delay)
                    
                    # Add jitter to prevent thundering herd
                    if jitter:
                        delay = delay * (0.5 + random.random())
                    
                    logger.warning(
                        f"[{func.__name__}] Attempt {attempt}/{max_attempts} failed: {e}. "
                        f"Retrying in {delay:.2f}s..."
                    )
                    
                    if on_retry:
                        on_retry(attempt, e, delay)
                    
                    time.sleep(delay)
            
            raise last_exception
        return wrapper
    return decorator

# Async version for FastAPI
import asyncio

def async_retry(max_attempts=3, base_delay=1.0, retryable_exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return await func(*args, **kwargs)
                except retryable_exceptions as e:
                    if attempt == max_attempts:
                        raise
                    delay = base_delay * (2 ** (attempt - 1)) * (0.5 + random.random())
                    await asyncio.sleep(delay)
        return wrapper
    return decorator

# Usage
@retry(
    max_attempts=3,
    base_delay=2.0,
    retryable_exceptions=(ConnectionError, TimeoutError),
    on_retry=lambda attempt, e, delay: print(f"Retry callback: attempt {attempt}")
)
def fetch_hcp_data(api_url):
    """Fetch data from external HCP API."""
    import requests
    response = requests.get(api_url, timeout=10)
    response.raise_for_status()
    return response.json()

@async_retry(max_attempts=3, retryable_exceptions=(ConnectionError,))
async def async_fetch_data(url):
    import aiohttp
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            return await resp.json()
```

**Interview tip:** Always mention jitter (prevents thundering herd). Explain why max_delay exists (cap the wait). Discuss which exceptions should NOT be retried (4xx client errors).
</details>

---

## Problem 9: Implement an Event-Sourced State Machine (Hard)

> Design a state machine for tracking ETL job status with full event history.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from datetime import datetime
from typing import List, Dict, Optional, Callable
from dataclasses import dataclass, field
from enum import Enum

class JobState(Enum):
    CREATED = "created"
    QUEUED = "queued"
    RUNNING = "running"
    PAUSED = "paused"
    SUCCEEDED = "succeeded"
    FAILED = "failed"
    RETRYING = "retrying"
    CANCELLED = "cancelled"

@dataclass
class Event:
    event_type: str
    timestamp: datetime
    data: Dict = field(default_factory=dict)
    actor: str = "system"

class InvalidTransitionError(Exception):
    pass

class ETLJobStateMachine:
    VALID_TRANSITIONS = {
        JobState.CREATED: [JobState.QUEUED, JobState.CANCELLED],
        JobState.QUEUED: [JobState.RUNNING, JobState.CANCELLED],
        JobState.RUNNING: [JobState.SUCCEEDED, JobState.FAILED, JobState.PAUSED, JobState.CANCELLED],
        JobState.PAUSED: [JobState.RUNNING, JobState.CANCELLED],
        JobState.FAILED: [JobState.RETRYING, JobState.CANCELLED],
        JobState.RETRYING: [JobState.QUEUED],
        JobState.SUCCEEDED: [],
        JobState.CANCELLED: [],
    }

    def __init__(self, job_id: str):
        self.job_id = job_id
        self.state = JobState.CREATED
        self.events: List[Event] = []
        self.listeners: Dict[str, List[Callable]] = {}
        self._record_event("job_created", {})

    def transition(self, new_state: JobState, data: Dict = None, actor: str = "system"):
        if new_state not in self.VALID_TRANSITIONS[self.state]:
            raise InvalidTransitionError(
                f"Cannot transition from {self.state.value} to {new_state.value}. "
                f"Valid transitions: {[s.value for s in self.VALID_TRANSITIONS[self.state]]}"
            )
        
        old_state = self.state
        self.state = new_state
        
        event = self._record_event(
            f"state_changed:{old_state.value}->{new_state.value}",
            {"old_state": old_state.value, "new_state": new_state.value, **(data or {})}
        )
        
        self._notify_listeners(event)
        return event

    def _record_event(self, event_type: str, data: Dict) -> Event:
        event = Event(
            event_type=event_type,
            timestamp=datetime.utcnow(),
            data=data
        )
        self.events.append(event)
        return event

    def _notify_listeners(self, event: Event):
        for listener in self.listeners.get(event.event_type, []):
            listener(self, event)
        for listener in self.listeners.get("*", []):
            listener(self, event)

    def on(self, event_type: str, handler: Callable):
        self.listeners.setdefault(event_type, []).append(handler)

    def get_history(self) -> List[Dict]:
        return [
            {"type": e.event_type, "timestamp": e.timestamp.isoformat(), "data": e.data}
            for e in self.events
        ]

    def get_duration(self) -> Optional[float]:
        start = next((e for e in self.events if "RUNNING" in e.event_type), None)
        end = next((e for e in reversed(self.events) 
                    if "SUCCEEDED" in e.event_type or "FAILED" in e.event_type), None)
        if start and end:
            return (end.timestamp - start.timestamp).total_seconds()
        return None

    @classmethod
    def replay(cls, job_id: str, events: List[Event]) -> 'ETLJobStateMachine':
        """Reconstruct state from event history (event sourcing)."""
        machine = cls(job_id)
        machine.events = []
        machine.state = JobState.CREATED
        
        for event in events:
            if event.event_type.startswith("state_changed:"):
                new_state_str = event.data["new_state"]
                machine.state = JobState(new_state_str)
            machine.events.append(event)
        
        return machine

# Usage
job = ETLJobStateMachine("etl-hcp-daily-001")
job.on("*", lambda sm, e: print(f"[{sm.job_id}] {e.event_type}"))

job.transition(JobState.QUEUED)
job.transition(JobState.RUNNING, {"dpu_count": 6})
# Simulate failure
job.transition(JobState.FAILED, {"error": "S3 timeout", "records_processed": 50000})
job.transition(JobState.RETRYING, {"attempt": 2})
job.transition(JobState.QUEUED)
job.transition(JobState.RUNNING, {"dpu_count": 8})
job.transition(JobState.SUCCEEDED, {"records_processed": 125000000})

print(f"\nDuration: {job.get_duration()}s")
print(f"Final state: {job.state.value}")
print(f"Event count: {len(job.events)}")
```

**Interview tip:** This demonstrates event sourcing — the ability to reconstruct any past state from events. Used in audit-heavy systems (healthcare, finance). Mention CQRS as a complementary pattern.
</details>

---

## Problem 10: Implement a Data Pipeline DAG Executor (Hard)

> Build a DAG-based pipeline executor where steps can run in parallel if no dependencies exist.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from collections import defaultdict, deque
from typing import Callable, Dict, List, Set
from dataclasses import dataclass
import threading
import time

@dataclass
class Task:
    name: str
    func: Callable
    dependencies: List[str]

class DAGExecutor:
    def __init__(self, max_workers: int = 4):
        self.tasks: Dict[str, Task] = {}
        self.max_workers = max_workers
        self.results: Dict[str, any] = {}
        self.errors: Dict[str, Exception] = {}
        self.lock = threading.Lock()

    def add_task(self, name: str, func: Callable, dependencies: List[str] = None):
        self.tasks[name] = Task(name=name, func=func, dependencies=dependencies or [])

    def _topological_sort(self) -> List[List[str]]:
        """Returns tasks grouped by execution level (tasks in same level can run in parallel)."""
        in_degree = {name: 0 for name in self.tasks}
        graph = defaultdict(list)
        
        for name, task in self.tasks.items():
            for dep in task.dependencies:
                graph[dep].append(name)
                in_degree[name] += 1
        
        # BFS level-order
        levels = []
        queue = deque([name for name, degree in in_degree.items() if degree == 0])
        
        while queue:
            level = []
            for _ in range(len(queue)):
                node = queue.popleft()
                level.append(node)
                for neighbor in graph[node]:
                    in_degree[neighbor] -= 1
                    if in_degree[neighbor] == 0:
                        queue.append(neighbor)
            levels.append(level)
        
        # Check for cycles
        if sum(len(l) for l in levels) != len(self.tasks):
            raise ValueError("Cycle detected in DAG!")
        
        return levels

    def execute(self) -> Dict[str, any]:
        levels = self._topological_sort()
        
        with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
            for level in levels:
                # All tasks in this level can run in parallel
                futures = {}
                for task_name in level:
                    task = self.tasks[task_name]
                    
                    # Check if any dependency failed
                    failed_deps = [d for d in task.dependencies if d in self.errors]
                    if failed_deps:
                        self.errors[task_name] = Exception(
                            f"Skipped: dependency {failed_deps} failed"
                        )
                        continue
                    
                    futures[executor.submit(task.func, self.results)] = task_name
                
                # Wait for all tasks in this level to complete
                for future in as_completed(futures):
                    task_name = futures[future]
                    try:
                        result = future.result()
                        with self.lock:
                            self.results[task_name] = result
                        print(f"✓ {task_name} completed")
                    except Exception as e:
                        with self.lock:
                            self.errors[task_name] = e
                        print(f"✗ {task_name} failed: {e}")
        
        return {"results": self.results, "errors": self.errors}

# Usage: ETL Pipeline
def extract_aurora(ctx):
    time.sleep(1)  # Simulate DB read
    ctx["aurora_data"] = [{"id": i, "name": f"record_{i}"} for i in range(1000)]
    return "Extracted 1000 records from Aurora"

def extract_s3(ctx):
    time.sleep(1.5)  # Simulate S3 read
    ctx["s3_data"] = [{"id": i, "file": f"file_{i}.csv"} for i in range(500)]
    return "Extracted 500 files from S3"

def transform_clean(ctx):
    time.sleep(0.5)
    data = ctx.get("aurora_data", []) + ctx.get("s3_data", [])
    ctx["clean_data"] = [r for r in data if r.get("id") is not None]
    return f"Cleaned {len(ctx['clean_data'])} records"

def transform_enrich(ctx):
    time.sleep(0.5)
    ctx["enriched_data"] = [
        {**r, "enriched": True} for r in ctx.get("clean_data", [])
    ]
    return f"Enriched {len(ctx['enriched_data'])} records"

def load_redshift(ctx):
    time.sleep(1)
    count = len(ctx.get("enriched_data", []))
    return f"Loaded {count} records to Redshift"

def load_elasticsearch(ctx):
    time.sleep(0.8)
    count = len(ctx.get("enriched_data", []))
    return f"Indexed {count} records in Elasticsearch"

def notify_completion(ctx):
    return "Notification sent: Pipeline completed"

# Build DAG
dag = DAGExecutor(max_workers=4)
dag.add_task("extract_aurora", extract_aurora)
dag.add_task("extract_s3", extract_s3)
dag.add_task("transform_clean", transform_clean, dependencies=["extract_aurora", "extract_s3"])
dag.add_task("transform_enrich", transform_enrich, dependencies=["transform_clean"])
dag.add_task("load_redshift", load_redshift, dependencies=["transform_enrich"])
dag.add_task("load_elasticsearch", load_elasticsearch, dependencies=["transform_enrich"])
dag.add_task("notify", notify_completion, dependencies=["load_redshift", "load_elasticsearch"])

# Execute
# extract_aurora and extract_s3 run in PARALLEL (no deps)
# transform_clean waits for both extracts
# load_redshift and load_elasticsearch run in PARALLEL
# notify waits for both loads
results = dag.execute()
```

**Execution visualization:**
```
Level 0: [extract_aurora, extract_s3]      ← Run in parallel
Level 1: [transform_clean]                  ← Waits for Level 0
Level 2: [transform_enrich]                 ← Waits for Level 1
Level 3: [load_redshift, load_elasticsearch] ← Run in parallel
Level 4: [notify]                           ← Waits for Level 3
```

**Interview tip:** This is how Airflow/Step Functions work internally. Topological sort determines execution order. Tasks at the same level have no dependencies on each other → parallelize. Discuss how to add: retry logic, timeout per task, checkpointing for resume.
</details>

---

## Summary — Problem Difficulty Map

| # | Problem | Difficulty | Topic | Time |
|---|---------|-----------|-------|------|
| 1 | Two Sum Streaming | Easy→Medium | DSA + Streaming | 10-15 min |
| 2 | Merge K Sorted | Medium | DSA + ETL | 15-20 min |
| 3 | Producer-Consumer Queue | Medium | Concurrency | 20 min |
| 4 | Trie Autocomplete | Medium | DSA + Search | 20 min |
| 5 | Duplicate Detection | Medium | Data Engineering | 15-20 min |
| 6 | Consistent Hashing | Hard | System Design | 25-30 min |
| 7 | Map-Reduce Framework | Hard | Distributed Systems | 30 min |
| 8 | Retry + Backoff | Easy | Production Patterns | 10 min |
| 9 | Event-Sourced State Machine | Hard | Architecture | 30 min |
| 10 | DAG Executor | Hard | Pipeline Orchestration | 35 min |

**Practice order:** 8 → 1 → 4 → 3 → 2 → 5 → 6 → 9 → 7 → 10
