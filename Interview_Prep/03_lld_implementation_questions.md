# LLD & Implementation Questions — Likely in Your Interview

## Part A: Low-Level Design Problems

---

### LLD-1: Design a Rate Limiter Class

> "Implement a rate limiter that allows N requests per time window."

```python
import time
from collections import defaultdict
from threading import Lock

class SlidingWindowRateLimiter:
    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests = defaultdict(list)  # user_id -> [timestamps]
        self.lock = Lock()

    def is_allowed(self, user_id: str) -> bool:
        with self.lock:
            now = time.time()
            window_start = now - self.window_seconds
            
            # Remove expired timestamps
            self.requests[user_id] = [
                ts for ts in self.requests[user_id] if ts > window_start
            ]
            
            if len(self.requests[user_id]) < self.max_requests:
                self.requests[user_id].append(now)
                return True
            return False

# Token Bucket variant (better for bursty traffic)
class TokenBucketRateLimiter:
    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.tokens = capacity
        self.last_refill = time.time()
        self.lock = Lock()

    def is_allowed(self) -> bool:
        with self.lock:
            now = time.time()
            elapsed = now - self.last_refill
            self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
            self.last_refill = now
            
            if self.tokens >= 1:
                self.tokens -= 1
                return True
            return False
```

**Follow-ups:** How to make it distributed? (Redis + Lua scripts). How to handle multiple rate limit tiers?

---

### LLD-2: Design an LRU Cache

> "Implement a Least Recently Used cache with O(1) get and put."

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key: str):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key: str, value) -> None:
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)

# From scratch (doubly linked list + hashmap)
class Node:
    def __init__(self, key=0, val=0):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None

class LRUCacheManual:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}
        self.head = Node()  # dummy head
        self.tail = Node()  # dummy tail
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev

    def _add_to_front(self, node):
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node

    def get(self, key):
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)
        self._add_to_front(node)
        return node.val

    def put(self, key, value):
        if key in self.cache:
            self._remove(self.cache[key])
        node = Node(key, value)
        self._add_to_front(node)
        self.cache[key] = node
        if len(self.cache) > self.capacity:
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]
```

---

### LLD-3: Design a Task Scheduler / Job Queue

> "Design a job scheduling system that can run tasks with priorities and retries."

```python
import heapq
import time
from enum import Enum
from dataclasses import dataclass, field
from typing import Callable, Any
from threading import Thread, Lock
from concurrent.futures import ThreadPoolExecutor

class JobStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    RETRYING = "retrying"

@dataclass(order=True)
class Job:
    priority: int
    scheduled_time: float
    job_id: str = field(compare=False)
    func: Callable = field(compare=False)
    args: tuple = field(default=(), compare=False)
    max_retries: int = field(default=3, compare=False)
    retry_count: int = field(default=0, compare=False)
    status: JobStatus = field(default=JobStatus.PENDING, compare=False)

class TaskScheduler:
    def __init__(self, max_workers: int = 4):
        self.queue = []  # min-heap
        self.lock = Lock()
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
        self.jobs = {}  # job_id -> Job
        self.running = True

    def submit(self, job_id: str, func: Callable, priority: int = 5,
               delay: float = 0, max_retries: int = 3, args: tuple = ()):
        job = Job(
            priority=priority,
            scheduled_time=time.time() + delay,
            job_id=job_id,
            func=func,
            args=args,
            max_retries=max_retries
        )
        with self.lock:
            heapq.heappush(self.queue, job)
            self.jobs[job_id] = job

    def _execute(self, job: Job):
        job.status = JobStatus.RUNNING
        try:
            job.func(*job.args)
            job.status = JobStatus.COMPLETED
        except Exception as e:
            if job.retry_count < job.max_retries:
                job.retry_count += 1
                job.status = JobStatus.RETRYING
                job.scheduled_time = time.time() + (2 ** job.retry_count)
                with self.lock:
                    heapq.heappush(self.queue, job)
            else:
                job.status = JobStatus.FAILED

    def run(self):
        while self.running:
            with self.lock:
                if self.queue and self.queue[0].scheduled_time <= time.time():
                    job = heapq.heappop(self.queue)
                    self.executor.submit(self._execute, job)
            time.sleep(0.1)

    def get_status(self, job_id: str) -> JobStatus:
        return self.jobs[job_id].status if job_id in self.jobs else None
```

**Follow-ups:** How to make it distributed? How to handle job deduplication? How to add cron-like scheduling?

---

### LLD-4: Design a Connection Pool

> "Implement a database connection pool for your ETL pipeline."

```python
import threading
from queue import Queue, Empty
from contextlib import contextmanager

class Connection:
    def __init__(self, host, port, db):
        self.host = host
        self.port = port
        self.db = db
        self.is_alive = True
    
    def execute(self, query):
        pass  # actual DB execution
    
    def close(self):
        self.is_alive = False

class ConnectionPool:
    def __init__(self, host, port, db, min_size=5, max_size=20, timeout=30):
        self.host = host
        self.port = port
        self.db = db
        self.min_size = min_size
        self.max_size = max_size
        self.timeout = timeout
        self.pool = Queue(maxsize=max_size)
        self.size = 0
        self.lock = threading.Lock()
        
        # Pre-create minimum connections
        for _ in range(min_size):
            self.pool.put(self._create_connection())
            self.size += 1

    def _create_connection(self) -> Connection:
        return Connection(self.host, self.port, self.db)

    def acquire(self) -> Connection:
        try:
            conn = self.pool.get(timeout=self.timeout)
            if not conn.is_alive:
                conn = self._create_connection()
            return conn
        except Empty:
            with self.lock:
                if self.size < self.max_size:
                    self.size += 1
                    return self._create_connection()
            raise TimeoutError("Connection pool exhausted")

    def release(self, conn: Connection):
        if conn.is_alive:
            self.pool.put(conn)
        else:
            with self.lock:
                self.size -= 1

    @contextmanager
    def get_connection(self):
        conn = self.acquire()
        try:
            yield conn
        finally:
            self.release(conn)

# Usage
pool = ConnectionPool("localhost", 5432, "analytics", min_size=5, max_size=20)
with pool.get_connection() as conn:
    conn.execute("SELECT * FROM hcp_records LIMIT 100")
```

---

### LLD-5: Design a Pub/Sub Event System

> "Design an in-process event bus for decoupling microservice components."

```python
from collections import defaultdict
from typing import Callable, Any
from concurrent.futures import ThreadPoolExecutor
import asyncio

class EventBus:
    def __init__(self):
        self._subscribers = defaultdict(list)
        self._executor = ThreadPoolExecutor(max_workers=4)

    def subscribe(self, event_type: str, handler: Callable, priority: int = 0):
        self._subscribers[event_type].append((priority, handler))
        self._subscribers[event_type].sort(key=lambda x: x[0], reverse=True)

    def unsubscribe(self, event_type: str, handler: Callable):
        self._subscribers[event_type] = [
            (p, h) for p, h in self._subscribers[event_type] if h != handler
        ]

    def publish(self, event_type: str, data: Any = None):
        for _, handler in self._subscribers[event_type]:
            self._executor.submit(handler, data)

    def publish_sync(self, event_type: str, data: Any = None):
        for _, handler in self._subscribers[event_type]:
            handler(data)

# Usage
bus = EventBus()

def on_record_processed(data):
    print(f"Record {data['id']} processed")

def on_record_failed(data):
    print(f"Record {data['id']} failed: {data['error']}")

bus.subscribe("record.processed", on_record_processed)
bus.subscribe("record.failed", on_record_failed)
bus.publish("record.processed", {"id": "HCP-001"})
```

---

## Part B: Live Coding / Implementation Questions

---

### IMPL-1: Implement a Data Pipeline Validator

> "Write a function that validates data quality in an ETL pipeline."

```python
from dataclasses import dataclass
from typing import List, Dict, Callable, Any
from enum import Enum

class Severity(Enum):
    ERROR = "error"      # Pipeline should stop
    WARNING = "warning"  # Log but continue
    INFO = "info"        # Informational

@dataclass
class ValidationResult:
    rule_name: str
    passed: bool
    severity: Severity
    message: str
    affected_rows: int = 0

class DataValidator:
    def __init__(self):
        self.rules: List[Dict] = []

    def add_rule(self, name: str, check: Callable, severity: Severity = Severity.ERROR):
        self.rules.append({"name": name, "check": check, "severity": severity})

    def validate(self, df) -> List[ValidationResult]:
        results = []
        for rule in self.rules:
            try:
                passed, message, affected = rule["check"](df)
                results.append(ValidationResult(
                    rule_name=rule["name"],
                    passed=passed,
                    severity=rule["severity"],
                    message=message,
                    affected_rows=affected
                ))
            except Exception as e:
                results.append(ValidationResult(
                    rule_name=rule["name"],
                    passed=False,
                    severity=Severity.ERROR,
                    message=f"Rule execution failed: {str(e)}"
                ))
        return results

    def should_halt(self, results: List[ValidationResult]) -> bool:
        return any(not r.passed and r.severity == Severity.ERROR for r in results)

# Usage
validator = DataValidator()
validator.add_rule(
    "no_null_ids",
    lambda df: (df["id"].notna().all(), "Null IDs found", df["id"].isna().sum()),
    Severity.ERROR
)
validator.add_rule(
    "valid_dates",
    lambda df: (df["date"].between("2020-01-01", "2026-12-31").all(), 
                "Invalid dates", (~df["date"].between("2020-01-01", "2026-12-31")).sum()),
    Severity.WARNING
)
```

---

### IMPL-2: Implement Retry with Circuit Breaker

> "Implement a circuit breaker pattern for external API calls."

```python
import time
from enum import Enum
from functools import wraps

class CircuitState(Enum):
    CLOSED = "closed"       # Normal operation
    OPEN = "open"           # Failing, reject calls
    HALF_OPEN = "half_open" # Testing if recovered

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30, success_threshold=3):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.success_threshold = success_threshold
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None

    def __call__(self, func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            if self.state == CircuitState.OPEN:
                if time.time() - self.last_failure_time > self.recovery_timeout:
                    self.state = CircuitState.HALF_OPEN
                else:
                    raise CircuitOpenError(f"Circuit is OPEN. Retry after {self.recovery_timeout}s")
            
            try:
                result = func(*args, **kwargs)
                self._on_success()
                return result
            except Exception as e:
                self._on_failure()
                raise
        return wrapper

    def _on_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state = CircuitState.CLOSED
                self.failure_count = 0
                self.success_count = 0

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
            self.success_count = 0

class CircuitOpenError(Exception):
    pass

# Usage
circuit = CircuitBreaker(failure_threshold=3, recovery_timeout=60)

@circuit
def call_external_api(endpoint, payload):
    response = requests.post(endpoint, json=payload, timeout=10)
    response.raise_for_status()
    return response.json()
```

---

### IMPL-3: Implement a Simple Workflow Engine

> "Design a lightweight workflow engine for ETL step orchestration."

```python
from enum import Enum
from typing import Callable, Dict, List, Optional
import time
import logging

class StepStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    SUCCESS = "success"
    FAILED = "failed"
    SKIPPED = "skipped"

class Step:
    def __init__(self, name: str, func: Callable, depends_on: List[str] = None,
                 retry_count: int = 0, skip_on_failure: bool = False):
        self.name = name
        self.func = func
        self.depends_on = depends_on or []
        self.retry_count = retry_count
        self.skip_on_failure = skip_on_failure
        self.status = StepStatus.PENDING
        self.result = None
        self.error = None

class Workflow:
    def __init__(self, name: str):
        self.name = name
        self.steps: Dict[str, Step] = {}
        self.context: Dict = {}
        self.logger = logging.getLogger(name)

    def add_step(self, name, func, depends_on=None, retry_count=0, skip_on_failure=False):
        self.steps[name] = Step(name, func, depends_on, retry_count, skip_on_failure)

    def _can_run(self, step: Step) -> bool:
        for dep in step.depends_on:
            if self.steps[dep].status != StepStatus.SUCCESS:
                return False
        return True

    def _should_skip(self, step: Step) -> bool:
        for dep in step.depends_on:
            if self.steps[dep].status == StepStatus.FAILED:
                return True
        return False

    def run(self) -> Dict[str, StepStatus]:
        pending = set(self.steps.keys())
        
        while pending:
            progressed = False
            for name in list(pending):
                step = self.steps[name]
                
                if self._should_skip(step) and step.skip_on_failure:
                    step.status = StepStatus.SKIPPED
                    pending.remove(name)
                    progressed = True
                    continue
                
                if self._can_run(step):
                    self._execute_step(step)
                    pending.remove(name)
                    progressed = True
            
            if not progressed:
                # Deadlock or all remaining have failed dependencies
                for name in pending:
                    self.steps[name].status = StepStatus.SKIPPED
                break
        
        return {name: step.status for name, step in self.steps.items()}

    def _execute_step(self, step: Step):
        step.status = StepStatus.RUNNING
        attempts = step.retry_count + 1
        
        for attempt in range(attempts):
            try:
                step.result = step.func(self.context)
                step.status = StepStatus.SUCCESS
                self.logger.info(f"Step '{step.name}' completed")
                return
            except Exception as e:
                if attempt < attempts - 1:
                    self.logger.warning(f"Step '{step.name}' failed (attempt {attempt+1}), retrying...")
                    time.sleep(2 ** attempt)
                else:
                    step.status = StepStatus.FAILED
                    step.error = str(e)
                    self.logger.error(f"Step '{step.name}' failed: {e}")

# Usage
def extract(ctx):
    ctx["raw_data"] = load_from_s3("s3://bucket/raw/")

def transform(ctx):
    ctx["clean_data"] = clean_and_validate(ctx["raw_data"])

def load(ctx):
    write_to_redshift(ctx["clean_data"])

pipeline = Workflow("daily_hcp_etl")
pipeline.add_step("extract", extract, retry_count=2)
pipeline.add_step("transform", transform, depends_on=["extract"])
pipeline.add_step("load", load, depends_on=["transform"], retry_count=1)
results = pipeline.run()
```

---

### IMPL-4: Implement WebSocket Chat Server (Based on Your Project)

> "Show me how you'd implement the core WebSocket handling for your chat app."

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict, Set
import json
from datetime import datetime

app = FastAPI()

class ConnectionManager:
    def __init__(self):
        self.active_connections: Dict[str, WebSocket] = {}
        self.rooms: Dict[str, Set[str]] = {}

    async def connect(self, websocket: WebSocket, user_id: str):
        await websocket.accept()
        self.active_connections[user_id] = websocket

    def disconnect(self, user_id: str):
        self.active_connections.pop(user_id, None)
        for room in self.rooms.values():
            room.discard(user_id)

    async def send_personal(self, user_id: str, message: dict):
        if ws := self.active_connections.get(user_id):
            await ws.send_json(message)

    async def broadcast_to_room(self, room_id: str, message: dict, exclude: str = None):
        for user_id in self.rooms.get(room_id, set()):
            if user_id != exclude:
                await self.send_personal(user_id, message)

    def join_room(self, user_id: str, room_id: str):
        self.rooms.setdefault(room_id, set()).add(user_id)

    def leave_room(self, user_id: str, room_id: str):
        if room_id in self.rooms:
            self.rooms[room_id].discard(user_id)

manager = ConnectionManager()

@app.websocket("/ws/{user_id}")
async def websocket_endpoint(websocket: WebSocket, user_id: str):
    await manager.connect(websocket, user_id)
    try:
        while True:
            data = await websocket.receive_json()
            
            if data["type"] == "join_room":
                manager.join_room(user_id, data["room_id"])
                await manager.broadcast_to_room(data["room_id"], {
                    "type": "user_joined",
                    "user_id": user_id,
                    "timestamp": datetime.utcnow().isoformat()
                })
            
            elif data["type"] == "message":
                await manager.broadcast_to_room(data["room_id"], {
                    "type": "message",
                    "user_id": user_id,
                    "content": data["content"],
                    "room_id": data["room_id"],
                    "timestamp": datetime.utcnow().isoformat()
                }, exclude=user_id)
            
            elif data["type"] == "typing":
                await manager.broadcast_to_room(data["room_id"], {
                    "type": "typing",
                    "user_id": user_id
                }, exclude=user_id)
    
    except WebSocketDisconnect:
        manager.disconnect(user_id)
```

---

### IMPL-5: Design a Caching Layer with TTL and Eviction

> "Implement a cache with TTL, max-size eviction, and cache-aside pattern."

```python
import time
from threading import Lock
from collections import OrderedDict
from typing import Optional, Callable, Any

class TTLCache:
    def __init__(self, max_size: int = 1000, default_ttl: int = 300):
        self.max_size = max_size
        self.default_ttl = default_ttl
        self.cache = OrderedDict()  # key -> (value, expire_time)
        self.lock = Lock()
        self.hits = 0
        self.misses = 0

    def get(self, key: str) -> Optional[Any]:
        with self.lock:
            if key in self.cache:
                value, expire_time = self.cache[key]
                if time.time() < expire_time:
                    self.cache.move_to_end(key)
                    self.hits += 1
                    return value
                else:
                    del self.cache[key]
            self.misses += 1
            return None

    def set(self, key: str, value: Any, ttl: int = None):
        with self.lock:
            expire_time = time.time() + (ttl or self.default_ttl)
            if key in self.cache:
                self.cache.move_to_end(key)
            self.cache[key] = (value, expire_time)
            
            while len(self.cache) > self.max_size:
                self.cache.popitem(last=False)

    def delete(self, key: str):
        with self.lock:
            self.cache.pop(key, None)

    def cache_aside(self, key: str, fetch_func: Callable, ttl: int = None) -> Any:
        value = self.get(key)
        if value is not None:
            return value
        value = fetch_func()
        self.set(key, value, ttl)
        return value

    @property
    def hit_rate(self) -> float:
        total = self.hits + self.misses
        return self.hits / total if total > 0 else 0.0

# Usage
cache = TTLCache(max_size=5000, default_ttl=600)

def get_hcp_record(hcp_id: str):
    return cache.cache_aside(
        key=f"hcp:{hcp_id}",
        fetch_func=lambda: db.query(f"SELECT * FROM hcp WHERE id = '{hcp_id}'"),
        ttl=300
    )
```

---

## Part C: Design Patterns They'll Ask About

| Pattern | When to Use | Your Resume Connection |
|---------|-------------|----------------------|
| **Strategy** | Multiple algorithms, runtime selection | Different ETL transformations per data source |
| **Observer** | Event-driven notifications | WebSocket broadcast in chat app |
| **Factory** | Creating objects without specifying class | Creating different DB connectors (Aurora, Redshift) |
| **Decorator** | Adding behavior without modifying class | Python decorators for retry, logging, caching |
| **Builder** | Complex object construction | SQL query builders, pipeline configuration |
| **Singleton** | Single instance (DB pool, config) | Connection pool, app configuration |
| **Command** | Encapsulate actions as objects | ETL step execution, undo operations |
| **Template Method** | Define algorithm skeleton | Base ETL class with customizable steps |

---

## Key Tips for Implementation Round

1. **Ask clarifying questions** before coding
2. **Start with interface/class structure** — show you think before coding
3. **Write clean, readable code** — use proper naming, type hints
4. **Handle edge cases** — empty inputs, concurrent access, failures
5. **Discuss time/space complexity** of your solution
6. **Mention testing approach** — "I'd test this with..."
7. **Relate to real experience** — "In my ETL pipeline, I used this pattern for..."
