# Operating Systems — Complete Deep Dive

## Why OS Matters in Interviews
Every FAANG company asks OS questions because it reveals whether you understand **how software actually runs**. System design, debugging, and performance optimization all require OS knowledge.

---

## 1. Process vs Thread

### Process
A **process** is an independent program in execution with its own memory space.

```
┌─────────────────────────────────────────────┐
│              Process Memory Layout           │
├─────────────────────────────────────────────┤
│  Stack      │ Function calls, local vars    │  ← grows downward
│  ↓          │                               │
│             │       (free space)            │
│  ↑          │                               │
│  Heap       │ Dynamic allocation (malloc)   │  ← grows upward
│  BSS        │ Uninitialized global vars     │
│  Data       │ Initialized global vars       │
│  Text/Code  │ Program instructions (read-only)│
└─────────────────────────────────────────────┘

Each process has:
- Unique PID (Process ID)
- Own virtual address space (isolated!)
- Own file descriptors, registers, stack
- PCB (Process Control Block) in kernel
```

### Thread
A **thread** is a lightweight unit of execution within a process. Threads **share** the process's memory.

```
┌────────────────── Process ──────────────────┐
│                                              │
│  Shared: Code, Data, Heap, File descriptors │
│                                              │
│  ┌────────┐   ┌────────┐   ┌────────┐     │
│  │Thread 1│   │Thread 2│   │Thread 3│     │
│  │Own stack│   │Own stack│   │Own stack│     │
│  │Own regs │   │Own regs │   │Own regs │     │
│  │Own PC   │   │Own PC   │   │Own PC   │     │
│  └────────┘   └────────┘   └────────┘     │
└──────────────────────────────────────────────┘
```

### Key Differences

```
┌────────────────────┬──────────────────────┬──────────────────────┐
│                    │ Process              │ Thread               │
├────────────────────┼──────────────────────┼──────────────────────┤
│ Memory             │ Separate address space│ Shared address space │
│ Creation cost      │ Heavy (fork + copy)  │ Light (share memory) │
│ Context switch     │ Slow (TLB flush)     │ Fast (same space)    │
│ Communication      │ IPC (pipes, sockets) │ Shared memory (easy) │
│ Crash isolation    │ Independent          │ One crash kills all  │
│ Use case           │ Chrome tabs, Docker  │ Web server workers   │
└────────────────────┴──────────────────────┴──────────────────────┘
```

### Process States
```
        ┌──────── NEW
        │
        ▼
    ┌───────┐   scheduled    ┌─────────┐
    │ READY │──────────────▶│ RUNNING │
    │       │◀──────────────│         │
    └───┬───┘   preempted   └────┬────┘
        │                        │
        │                   I/O or wait
        │                        │
        │                   ┌────▼─────┐
        │                   │ WAITING  │
        │                   │(BLOCKED) │
        │                   └────┬─────┘
        │                        │
        └──── I/O complete ──────┘
                                 │
                            ┌────▼────┐
                            │TERMINATED│
                            └──────────┘
```

---

## 2. Process Scheduling

### Scheduling Criteria
- **CPU Utilization** — Keep CPU busy (target: 40-90%)
- **Throughput** — Processes completed per time unit
- **Turnaround Time** — Total time from submission to completion
- **Waiting Time** — Time spent in ready queue
- **Response Time** — Time from request to first response

### Scheduling Algorithms

**1. FCFS (First Come, First Served)**
```
Process  Burst Time
P1       24ms
P2       3ms
P3       3ms

Order: P1 → P2 → P3
Waiting: P1=0, P2=24, P3=27
Average waiting = (0+24+27)/3 = 17ms

⚠️ Convoy Effect: Short processes wait behind long ones
```

**2. SJF (Shortest Job First)**
```
Optimal for average waiting time (provably!)
Non-preemptive: Run shortest job next
Preemptive (SRTF): Switch to shorter job if one arrives

Problem: How to predict burst time? (use exponential averaging)
τ(n+1) = α × t(n) + (1-α) × τ(n)  where α = 0.5 typically
```

**3. Round Robin (RR)**
```
Each process gets a fixed time quantum (e.g., 20ms)
After quantum expires → move to back of queue

Time Quantum too large → becomes FCFS
Time Quantum too small → too much context switching overhead
Sweet spot: 80% of CPU bursts should be shorter than quantum

Used by: Linux CFS (with dynamic priorities)
```

**4. Priority Scheduling**
```
Each process has priority. Higher priority runs first.
Problem: Starvation (low priority never runs)
Solution: Aging (increase priority over time)
```

**5. Multilevel Feedback Queue (Modern OS)**
```
Queue 0: [Highest priority, short quantum (8ms)]    ← Interactive
Queue 1: [Medium priority, longer quantum (16ms)]   ← I/O bound  
Queue 2: [Lowest priority, FCFS]                    ← CPU bound

Rules:
- New process enters Queue 0
- If it uses full quantum → demote to Queue 1
- If it blocks for I/O → stays in current queue
- Periodically boost all processes to Queue 0 (prevent starvation)
```

---

## 3. Memory Management

### Virtual Memory
```
Each process sees its own virtual address space (0 to 2^64).
MMU (Memory Management Unit) translates virtual → physical.

Virtual Address → [Page Table] → Physical Address

Why virtual memory?
1. Isolation: Process A can't access Process B's memory
2. Illusion: Each process thinks it has all the memory
3. Efficiency: Only load pages that are actually used
4. Sharing: Multiple processes can share code pages (libraries)
```

### Paging
```
Virtual memory divided into fixed-size PAGES (typically 4KB)
Physical memory divided into FRAMES (same size)
Page table maps: page number → frame number

Virtual Address: [Page Number | Offset]
                      │              │
                 Page Table      Same offset
                      │              │
Physical Address: [Frame Number | Offset]

Multi-level page tables (saves memory):
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Page Dir │────▶│Page Table│────▶│  Frame   │
│  (L1)    │     │   (L2)   │     │(Physical)│
└──────────┘     └──────────┘     └──────────┘
```

### TLB (Translation Lookaside Buffer)
```
Cache for page table entries. CRITICAL for performance.

CPU → TLB check (1 cycle)
  HIT → Physical address immediately
  MISS → Walk page table (100+ cycles) → Update TLB

TLB hit ratio: 99%+ for most programs (locality!)
Context switch → TLB flush (expensive! why thread > process)
```

### Page Replacement Algorithms
```
When all frames full and page fault occurs → which page to evict?

1. FIFO: Replace oldest page
   ⚠️ Belady's Anomaly (more frames → more faults possible)

2. LRU (Least Recently Used): Replace page not used longest
   ✅ Good performance, approximates OPT
   Implementation: Counter per page (expensive) or Stack

3. Optimal (OPT): Replace page not needed for longest time
   Impossible in practice (requires future knowledge)
   Used as benchmark to compare other algorithms

4. Clock (Second Chance): FIFO + reference bit
   If reference bit = 1 → give second chance (set to 0, move on)
   If reference bit = 0 → replace this page
   ✅ Approximates LRU with low overhead (used in real OS)
```

### Thrashing
```
When system spends more time paging than executing.

Cause: Too many processes → each has too few frames → constant page faults
Solution: Working Set Model — give each process enough frames for its active pages
Detection: If page fault rate > threshold → reduce degree of multiprogramming
```

---

## 4. Deadlock

### Four Necessary Conditions (ALL must hold)
```
1. Mutual Exclusion    — Resource held by only one process at a time
2. Hold and Wait       — Process holds resources while waiting for more
3. No Preemption       — Resources can't be forcibly taken away
4. Circular Wait       — Cycle of processes each waiting for the next
```

### Deadlock Handling
```
Prevention:  Break one of the 4 conditions
  - Break circular wait: Order all resources, request in order only
  - Break hold & wait: Request ALL resources at once before starting

Avoidance:   Don't enter unsafe state (Banker's Algorithm)
  - Before granting request: "If I give this, can everyone still finish?"
  - If yes → grant. If no → wait.

Detection:   Allow deadlock, then detect and recover
  - Resource Allocation Graph (cycle = deadlock)
  - Recovery: Kill a process or preempt a resource

Ignorance:   Do nothing (Linux, Windows — reboot if stuck!)
  - Most practical: deadlocks are rare, prevention is expensive
```

### Banker's Algorithm (Know This!)
```
Given: Available resources, Max needs, Current allocation

Safe sequence exists? → System is SAFE (no deadlock possible)

Step 1: Find process whose remaining need ≤ Available
Step 2: Pretend it finishes (release its resources)
Step 3: Repeat until all processes finish (SAFE) or stuck (UNSAFE)
```

---

## 5. Synchronization

### Race Condition
```
Two threads accessing shared data without synchronization.
Result depends on execution order → unpredictable bugs!

Thread A: counter = counter + 1
Thread B: counter = counter + 1

Assembly:
  A: LOAD counter (=5)
  B: LOAD counter (=5)     ← B reads BEFORE A writes
  A: ADD 1 (=6)
  B: ADD 1 (=6)            ← B also gets 6, not 7!
  A: STORE counter (=6)
  B: STORE counter (=6)    ← Lost update! Should be 7!
```

### Critical Section Problem
```
Requirements for a solution:
1. Mutual Exclusion  — Only one thread in CS at a time
2. Progress          — If CS is free, waiting thread can enter
3. Bounded Waiting   — No thread waits forever (no starvation)
```

### Synchronization Primitives

**Mutex (Mutual Exclusion Lock)**
```python
mutex = Lock()

mutex.acquire()     # blocks if already locked
# critical section (only one thread here)
mutex.release()     # unlocks, another thread can enter
```

**Semaphore**
```python
# Counting semaphore — allows N threads
sem = Semaphore(3)  # allows 3 concurrent access

sem.acquire()  # decrement count (blocks if count = 0)
# critical section
sem.release()  # increment count

# Binary semaphore (count = 1) ≈ Mutex
```

**Monitor (Java synchronized, Python with)**
```
Higher-level: Mutex + Condition Variables bundled together
Only one thread can execute monitor's methods at a time

condition.wait()    — release lock, sleep until signaled
condition.signal()  — wake one waiting thread
condition.broadcast() — wake ALL waiting threads
```

### Classic Problems

**Producer-Consumer (Bounded Buffer)**
```python
buffer = []
empty = Semaphore(BUFFER_SIZE)  # empty slots
full = Semaphore(0)              # filled slots
mutex = Lock()

def producer():
    empty.acquire()      # wait for empty slot
    mutex.acquire()      # enter critical section
    buffer.append(item)
    mutex.release()
    full.release()       # signal consumer

def consumer():
    full.acquire()       # wait for item
    mutex.acquire()
    item = buffer.pop(0)
    mutex.release()
    empty.release()      # signal producer
```

**Reader-Writer Problem**
```
Multiple readers can read simultaneously (no conflict)
Writer needs exclusive access (no readers or other writers)

Solution: Read lock (shared) vs Write lock (exclusive)
Implementation: RWLock / ReadWriteLock
```

**Dining Philosophers**
```
5 philosophers, 5 forks, each needs 2 forks to eat.
Naive: Each picks up left fork → all waiting for right → DEADLOCK!

Solutions:
1. Pick up both forks atomically (mutex around both)
2. Odd philosophers pick left first, even pick right first
3. Allow at most 4 philosophers to try simultaneously
```

---

## 6. File Systems

```
Key Concepts:
─────────────
Inode:        Metadata (size, permissions, pointers to data blocks)
              Does NOT contain filename (filename is in directory)

Directory:    Special file mapping filename → inode number

Hard link:    Multiple filenames → same inode (same file, different names)
Soft link:    Special file containing PATH to target (like shortcut)

Journaling:   Write operations to log FIRST → then apply
              On crash: replay journal → consistent state (ext4, NTFS)

File Allocation:
  Contiguous:  Fast read, fragmentation problem
  Linked list: No fragmentation, slow random access
  Indexed:     Inode has block pointers (ext4, NTFS — hybrid approach)
```

---

## 7. Interview Questions (Top 20)

1. What happens when you type `./program` in terminal? (fork → exec → wait)
2. Difference between process and thread?
3. What is a context switch? What gets saved?
4. Explain virtual memory and page tables.
5. What is thrashing? How to prevent it?
6. Explain deadlock conditions and how to prevent.
7. What is a race condition? Give example and fix.
8. Mutex vs Semaphore — when to use which?
9. Explain producer-consumer problem and solution.
10. What is priority inversion? (Mars Pathfinder bug!)
11. How does `malloc` work internally?
12. Explain fork() — what does child get?
13. What are zombie and orphan processes?
14. Explain copy-on-write (COW) optimization.
15. How does the CPU cache affect multithreading? (false sharing)
16. What is a spinlock? When is it better than mutex?
17. Explain the difference between user-space and kernel-space threads.
18. What is DMA (Direct Memory Access)?
19. How does the Linux scheduler (CFS) work?
20. Explain memory-mapped files (mmap).
