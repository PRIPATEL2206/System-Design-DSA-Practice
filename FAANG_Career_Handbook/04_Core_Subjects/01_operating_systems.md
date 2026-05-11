# Operating Systems — What FAANG Interviews Actually Ask

## Why OS Matters
OS questions appear in system design (threading models, memory management) and dedicated rounds at companies like Google, Microsoft, and Amazon. You need solid fundamentals, not textbook depth.

---

## Processes vs Threads

### Process
- Independent execution unit with its own memory space
- Created via fork() (Unix) or CreateProcess() (Windows)
- Expensive to create and context-switch
- Isolated: one process crashing doesn't affect others

### Thread
- Lightweight execution unit within a process
- Shares memory space with other threads in same process
- Cheap to create, fast context switch
- Shared memory = easier communication BUT synchronization needed

### Key Comparison
| Aspect | Process | Thread |
|--------|---------|--------|
| Memory | Separate address space | Shared address space |
| Creation cost | Heavy (fork) | Light |
| Communication | IPC (pipes, sockets, shared memory) | Direct memory access |
| Isolation | Strong (crash isolated) | Weak (one thread can corrupt shared state) |
| Context switch | Expensive (TLB flush) | Cheap |

### Interview Question: "When to use processes vs threads?"
- **Processes:** When isolation is critical (microservices, browser tabs)
- **Threads:** When tasks need shared state and are CPU-bound or I/O-bound

---

## Concurrency & Synchronization

### The Problem: Race Condition
```python
# Two threads incrementing counter
counter = 0
# Thread 1: read counter (0), add 1, write (1)
# Thread 2: read counter (0), add 1, write (1)
# Expected: 2, Actual: 1 — RACE CONDITION
```

### Solutions

**Mutex (Mutual Exclusion):**
```python
import threading
lock = threading.Lock()
counter = 0

def increment():
    global counter
    lock.acquire()
    counter += 1
    lock.release()
    # Or use: with lock: counter += 1
```

**Semaphore:** Like a mutex but allows N threads (useful for connection pools)
```python
# Allow max 5 concurrent database connections
db_semaphore = threading.Semaphore(5)
```

**Condition Variable:** Wait until some condition is met
```python
condition = threading.Condition()
queue = []

def producer():
    with condition:
        queue.append(item)
        condition.notify()  # Wake up a waiting consumer

def consumer():
    with condition:
        while not queue:
            condition.wait()  # Release lock and sleep until notified
        item = queue.pop(0)
```

### Deadlock
Four conditions (ALL must be true for deadlock):
1. **Mutual Exclusion:** Resource held exclusively
2. **Hold and Wait:** Thread holds one resource while waiting for another
3. **No Preemption:** Resources can't be forcibly taken
4. **Circular Wait:** A→B→C→A dependency cycle

**Prevention:** Break any one condition. Most practical: **order lock acquisition** (always acquire locks in same global order).

### The Dining Philosophers (Classic Interview Problem)
```
5 philosophers, 5 forks. Each needs 2 forks to eat.
Solution: Number the forks. Always pick up lower-numbered fork first.
This prevents circular wait → no deadlock.
```

---

## Memory Management

### Virtual Memory
- Each process gets its own virtual address space (illusion of full memory)
- MMU (Memory Management Unit) translates virtual → physical
- Allows: isolation between processes, memory larger than physical RAM

### Paging
- Memory divided into fixed-size pages (typically 4KB)
- Page table maps virtual page → physical frame
- **Page fault:** Access to page not in RAM → OS loads from disk

### Page Replacement Algorithms
| Algorithm | Strategy | Pro | Con |
|-----------|----------|-----|-----|
| LRU | Evict least recently used | Good hit rate | Expensive to track exactly |
| FIFO | Evict oldest page | Simple | Belady's anomaly |
| Clock (Second Chance) | FIFO with reference bit | Approximates LRU, cheap | Less accurate |
| LFU | Evict least frequently used | Good for hot-cold data | Slow to adapt |

### Memory Layout of a Process
```
High Address ┌──────────────────┐
             │      Stack       │ ← Local variables, function calls (grows down)
             │        ↓         │
             │   (free space)   │
             │        ↑         │
             │       Heap       │ ← Dynamic allocation (malloc/new) (grows up)
             │       BSS        │ ← Uninitialized global variables
             │      Data        │ ← Initialized global variables
             │      Text        │ ← Program code (read-only)
Low Address  └──────────────────┘
```

### Stack vs Heap
| Stack | Heap |
|-------|------|
| Automatic allocation/deallocation | Manual (or garbage collected) |
| Fixed size (8MB typical) | Grows as needed |
| Fast (pointer increment) | Slower (fragmentation, allocation search) |
| Thread-local | Shared across threads |

---

## CPU Scheduling

### Common Algorithms
| Algorithm | Description | Used In |
|-----------|-------------|---------|
| FCFS | First Come First Served | Simple batch systems |
| SJF | Shortest Job First | Optimal avg wait (but unpractical — can't predict) |
| Round Robin | Time quantum per process | Interactive systems (Linux CFS approximates this) |
| Priority | Higher priority runs first | Real-time systems |
| Multilevel Queue | Different queues for different types | Linux (interactive vs batch) |

### Context Switch
What happens when OS switches between threads/processes:
1. Save current thread's registers, program counter, stack pointer
2. Update PCB (Process Control Block)
3. Load new thread's state
4. **Cost:** ~1-10 microseconds. Seems small, but adds up at high concurrency.

---

## I/O Models (Critical for System Design)

### Blocking I/O
Thread waits until I/O completes. Simple but wastes threads.

### Non-blocking I/O
Thread checks if I/O is ready, returns immediately if not. Requires polling.

### I/O Multiplexing (select/poll/epoll)
One thread monitors MANY file descriptors. When any is ready, process it.
```
epoll (Linux): Handles 100K+ concurrent connections efficiently
This is how Nginx, Redis, Node.js work — event loop + epoll
```

### Async I/O
Kernel handles I/O completely, notifies when done. Most scalable.

### Interview Application
"How does a web server handle 10K concurrent connections?"
- **Thread-per-connection:** 10K threads = too expensive (2MB stack each = 20GB)
- **Event-driven (epoll):** Single thread handles all connections, dispatches when ready
- **This is why Node.js/Nginx scale better than Apache for concurrent connections**

---

## File Systems (Brief)

Key concepts to know:
- **Inode:** Metadata about a file (permissions, size, block pointers). NOT the name.
- **File descriptor:** Integer handle to an open file
- **Hard link vs Soft link:** Hard link = another name pointing to same inode. Soft link = points to a path (can break).
- **Journaling:** Write-ahead log for crash recovery (ext4, NTFS)

---

## Common Interview Questions

**Q: What happens when you type a URL in browser?**
1. DNS resolution (check cache → recursive query → get IP)
2. TCP 3-way handshake (SYN → SYN-ACK → ACK)
3. TLS handshake (if HTTPS)
4. HTTP request sent
5. Server processes, returns response
6. Browser renders HTML/CSS/JS

**Q: What's a zombie process?**
Process that finished but parent hasn't called wait() yet. Entry stays in process table.

**Q: What's thrashing?**
System spends more time swapping pages than executing. Happens when working set > physical memory.

**Q: Difference between mutex and semaphore?**
- Mutex: Binary lock, only the thread that locked can unlock
- Semaphore: Counter, any thread can signal. Used for resource pools (e.g., max 5 DB connections)

**Q: What is a system call?**
Interface between user space and kernel. Examples: open(), read(), write(), fork(), exec(). Involves mode switch (user → kernel mode).

---

## What Matters Most for FAANG

1. **Processes vs Threads** — When to use each, trade-offs
2. **Concurrency primitives** — Mutex, semaphore, deadlock conditions + prevention
3. **Memory model** — Stack vs heap, virtual memory, page faults
4. **I/O models** — How servers handle concurrency (this connects directly to system design)
5. **The "What happens when..." question** — End-to-end system understanding
