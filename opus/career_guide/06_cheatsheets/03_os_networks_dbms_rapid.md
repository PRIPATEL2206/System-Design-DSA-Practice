# OS + Networks + DBMS — Rapid Fire (Top 100 Q&A)

## Operating Systems (35 Questions)

**1. Process vs Thread?**
Process: separate memory space, heavy. Thread: shared memory, lightweight.

**2. What is a context switch?**
Saving current process state (registers, PC, stack pointer) and loading another's. Expensive (~1-10μs).

**3. What is virtual memory?**
Illusion that each process has full address space. MMU maps virtual → physical via page tables.

**4. What is a page fault?**
Access to page not in physical memory → OS loads it from disk → very slow (~10ms).

**5. What is thrashing?**
CPU spending more time swapping pages than executing. Fix: reduce multiprogramming or increase RAM.

**6. Deadlock conditions?**
Mutual exclusion + Hold & wait + No preemption + Circular wait. All 4 must hold.

**7. Mutex vs Semaphore?**
Mutex: binary (1 thread), ownership concept. Semaphore: counting (N threads), no ownership.

**8. What is a race condition?**
Two threads access shared data without sync → result depends on execution order.

**9. What is priority inversion?**
High-priority thread waits for low-priority thread holding a lock. Fix: priority inheritance.

**10. fork() returns what?**
0 to child, child's PID to parent. Child gets copy of parent's address space (COW).

**11. Zombie process?**
Child terminated but parent hasn't called wait(). Entry remains in process table.

**12. Copy-on-Write (COW)?**
After fork(), pages shared until one writes → then copy. Saves memory for fork+exec.

**13. What is DMA?**
Direct Memory Access — device transfers data to/from memory without CPU involvement.

**14. User mode vs Kernel mode?**
User: restricted (can't access hardware). Kernel: privileged (full access). System call transitions.

**15. What is a spinlock?**
Busy-wait lock. Good for very short critical sections (avoids context switch overhead).

**16. Explain paging vs segmentation.**
Paging: fixed-size pages (no external fragmentation). Segmentation: variable-size logical units.

**17. What is TLB?**
Translation Lookaside Buffer — cache for page table entries. Hit = 1 cycle, miss = 100+ cycles.

**18. LRU vs FIFO page replacement?**
LRU: evict least recently used (good but expensive to track). FIFO: evict oldest (simple but Belady's anomaly).

**19. What causes a segmentation fault?**
Accessing memory outside process's valid address space (null pointer, buffer overflow).

**20. Explain inode.**
Metadata structure: file size, permissions, timestamps, pointers to data blocks. Does NOT contain filename.

---

## Computer Networks (35 Questions)

**21. TCP 3-way handshake?**
SYN → SYN-ACK → ACK. Synchronizes sequence numbers both ways.

**22. Why not 2-way handshake?**
Can't confirm both sides' sequence numbers. Old connection requests could be accepted.

**23. TCP vs UDP?**
TCP: reliable, ordered, connection-oriented. UDP: fast, no guarantee, connectionless.

**24. What is HTTP?**
Application layer protocol. Request-response. Stateless. Uses TCP (HTTP/3 uses QUIC/UDP).

**25. HTTP status codes?**
2xx: success. 3xx: redirect. 4xx: client error. 5xx: server error.

**26. What is DNS?**
Maps domain names to IP addresses. Hierarchy: Root → TLD → Authoritative.

**27. How does HTTPS work?**
TLS handshake: verify certificate → key exchange → symmetric encryption for data.

**28. GET vs POST?**
GET: read (idempotent, cached, in URL). POST: write (not idempotent, in body).

**29. What is a cookie?**
Small data stored in browser, sent with every request to same domain. Used for sessions.

**30. What is CORS?**
Cross-Origin Resource Sharing. Browser blocks requests to different domains unless server allows.

**31. WebSocket vs HTTP?**
HTTP: request-response (half-duplex). WebSocket: persistent bidirectional (full-duplex).

**32. What is NAT?**
Network Address Translation. Maps private IPs to public IP. Saves IPv4 addresses.

**33. What is a subnet mask?**
Divides IP into network and host portions. /24 = 255.255.255.0 = 256 addresses.

**34. TCP congestion control?**
Slow start (exponential) → Congestion avoidance (linear) → Fast recovery (on loss).

**35. What is a CDN?**
Content Delivery Network. Caches content at edge locations near users. Reduces latency.

**36. What is ARP?**
Address Resolution Protocol. Maps IP address → MAC address on local network.

**37. What is DHCP?**
Dynamic Host Configuration Protocol. Auto-assigns IP address to devices on network.

**38. HTTP/2 vs HTTP/1.1?**
HTTP/2: multiplexing, header compression, binary framing. No head-of-line blocking at HTTP level.

**39. What is a reverse proxy?**
Sits in front of servers. Routes requests, SSL termination, caching, load balancing. (Nginx)

**40. OSI layers (simplified)?**
Application → Transport (TCP/UDP) → Network (IP) → Data Link (Ethernet) → Physical (wires).

---

## DBMS (30 Questions)

**41. What is ACID?**
Atomicity (all/nothing), Consistency (valid states), Isolation (no interference), Durability (survives crash).

**42. What is normalization?**
Reduce redundancy. 1NF: atomic values. 2NF: no partial dependency. 3NF: no transitive dependency.

**43. When to denormalize?**
Read-heavy workloads, analytics queries, when JOIN cost > redundancy cost.

**44. What is an index?**
Data structure (B-Tree) for fast lookup. O(log n) instead of O(n). Trade: faster reads, slower writes.

**45. Clustered vs Non-clustered index?**
Clustered: data physically sorted by index (one per table, usually PK). Non-clustered: separate structure pointing to data.

**46. What is a transaction?**
Unit of work that is atomic. Either all statements succeed (COMMIT) or all fail (ROLLBACK).

**47. Isolation levels?**
Read Uncommitted → Read Committed → Repeatable Read → Serializable. Higher = safer but slower.

**48. What is a dirty read?**
Reading data that another transaction hasn't committed yet. Prevented from Read Committed up.

**49. What is MVCC?**
Multi-Version Concurrency Control. Each transaction sees a snapshot. Writers don't block readers.

**50. SQL JOIN types?**
INNER (both match), LEFT (all left + matching right), RIGHT, FULL OUTER, CROSS (cartesian).

**51. What is a foreign key?**
Column referencing another table's primary key. Enforces referential integrity.

**52. What is a view?**
Virtual table from a query. Materialized view = pre-computed and stored (fast but needs refresh).

**53. What is sharding?**
Splitting data across multiple databases by a key (user_id, region). Horizontal scaling.

**54. What is replication?**
Copying data to multiple servers. Master-slave: writes to master, reads from replicas.

**55. What is a deadlock in DBMS?**
Two transactions each waiting for a lock held by the other. DB detects and kills one.

**56. Optimistic vs Pessimistic locking?**
Optimistic: check version at commit (retry on conflict). Pessimistic: lock rows immediately (block others).

**57. What is CAP theorem?**
Can only guarantee 2 of 3: Consistency, Availability, Partition Tolerance. Partition always happens → choose CP or AP.

**58. SQL vs NoSQL?**
SQL: structured, ACID, complex queries. NoSQL: flexible schema, horizontal scale, simple access patterns.

**59. What is a stored procedure?**
Pre-compiled SQL code stored in database. Executes on server side. Reduces network roundtrips.

**60. What is connection pooling?**
Reuse database connections instead of creating new ones per request. Creating connection ≈ 50-100ms.

---

## Quick Recall Tips

```
Before answering:
1. Take 3 seconds to organize your thoughts
2. Start with a one-line definition
3. Give a real-world example
4. Mention trade-offs if applicable
5. Relate to system design if relevant

If you don't know:
"I'm not 100% sure about the specifics of [X], but from what I 
understand, it's related to [broader concept]. In practice, I would 
[practical approach]."
```
