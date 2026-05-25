# Module 5: Deadlocks

## 🧠 What You'll Learn
- What causes deadlocks (the four conditions)
- How to detect deadlocks (resource allocation graphs, algorithms)
- How to prevent deadlocks (breaking the four conditions)
- How to avoid deadlocks (Banker's algorithm)
- Deadlock recovery strategies
- Livelock and priority inversion

---

## 5.1 What is a Deadlock?

A **deadlock** is a state where a set of processes are each waiting for a resource held by another process in the set — **nobody can make progress**.

```
Classic example: Two threads, two locks

Thread A:                Thread B:
lock(mutex1);            lock(mutex2);
lock(mutex2); ← WAIT     lock(mutex1); ← WAIT
   │                        │
   └────── Both waiting for each other ──────┘
           DEADLOCK! 🔒
```

Real-world analogy:
```
    ┌─────────┐                ┌─────────┐
    │  Car A  │ ──wants──→     │  Car B  │
    │ (holds  │                │ (holds  │
    │  road   │  ←──wants──    │  road   │
    │  east)  │                │  north) │
    └─────────┘                └─────────┘
    
    Neither can move. Both must back up for either to proceed.
```

---

## 5.2 Four Necessary Conditions (Coffman Conditions)

**ALL FOUR** must hold simultaneously for a deadlock to occur:

| # | Condition | Description | Example |
|---|-----------|-------------|---------|
| 1 | **Mutual Exclusion** | Resource can only be used by one process | Printer, mutex lock |
| 2 | **Hold and Wait** | Process holds resource(s) while waiting for more | Thread holds lock A, wants lock B |
| 3 | **No Preemption** | Resources can't be forcibly taken away | Can't yank a mutex from a thread |
| 4 | **Circular Wait** | P1→P2→P3→...→P1 cycle of waiting | Thread A waits for B, B waits for A |

> **Key insight**: If you break ANY ONE condition, deadlock is impossible.

---

## 5.3 Resource Allocation Graph (RAG)

A visual tool for modeling and detecting deadlocks.

```
Notation:
  ○  = Process
  □  = Resource type (dots inside = instances)
  ○ ──→ □  = Request edge (process wants resource)
  □ ──→ ○  = Assignment edge (resource held by process)
```

### Example 1: No Deadlock
```
    ┌───┐         ┌───┐
    │ R1├────────→│P1 │
    │ •─│         └───┘
    └───┘           │
      ▲             │ (P1 requests R2)
      │             ▼
    ┌───┐         ┌───┐
    │P2 │←────────┤ R2│
    └───┘         │ • │
                  └───┘
    
    P1 holds R1, wants R2
    P2 holds R2
    P2 can finish → release R2 → P1 gets R2 → No deadlock
```

### Example 2: Deadlock!
```
    ┌───┐         ┌───┐
    │ R1├────────→│P1 │──────┐
    │ • │         └───┘      │ (wants R2)
    └───┘                    ▼
      ▲                   ┌───┐
      │                   │ R2│
      │ (wants R1)        │ • │
      │                   └─┬─┘
    ┌───┐                   │
    │P2 │←──────────────────┘
    └───┘
    
    P1 holds R1, wants R2
    P2 holds R2, wants R1
    CYCLE → DEADLOCK!
```

### Rule for Single-Instance Resources
- **Cycle in RAG → Deadlock** (guaranteed)

### Rule for Multi-Instance Resources
- **Cycle in RAG → Possible deadlock** (not guaranteed)
- Need the Banker's algorithm to confirm

---

## 5.4 Deadlock Detection

### For Single-Instance Resources: Wait-For Graph

Simplify the RAG by removing resource nodes:

```
RAG:                          Wait-For Graph:
P1 ──→ R1 ──→ P2            P1 ──→ P2
P2 ──→ R2 ──→ P1            P2 ──→ P1
                                 ↑ CYCLE = DEADLOCK
```

**Detection**: Run cycle detection (DFS) on the wait-for graph. O(V + E).

### For Multi-Instance Resources: Detection Algorithm

Similar to Banker's algorithm but checks current state:

```
Given:
  n = number of processes
  m = number of resource types
  
  Available[m]     = currently available resources
  Allocation[n][m] = what each process currently holds
  Request[n][m]    = what each process currently wants

Algorithm:
  1. Work = Available
  2. For each process Pi:
     If Request[i] <= Work:
        Work = Work + Allocation[i]  // Pi can finish, release resources
        Mark Pi as finished
  3. Repeat step 2 until no more processes can finish
  4. Any unmarked process is DEADLOCKED
```

### Worked Example

```
        Allocation    Request    Available
        A  B  C      A  B  C    A  B  C
P0      0  1  0      0  0  0    0  0  0
P1      2  0  0      2  0  2
P2      3  0  3      0  0  0
P3      2  1  1      1  0  0
P4      0  0  2      0  0  2

Step 1: Work = [0,0,0]
Step 2: Check P0: Request[0]=[0,0,0] <= [0,0,0]? YES
        Work = [0,0,0] + [0,1,0] = [0,1,0]
Step 3: Check P2: Request[2]=[0,0,0] <= [0,1,0]? YES
        Work = [0,1,0] + [3,0,3] = [3,1,3]
Step 4: Check P1: Request[1]=[2,0,2] <= [3,1,3]? YES
        Work = [3,1,3] + [2,0,0] = [5,1,3]
Step 5: Check P3: Request[3]=[1,0,0] <= [5,1,3]? YES
        Work = [5,1,3] + [2,1,1] = [7,2,4]
Step 6: Check P4: Request[4]=[0,0,2] <= [7,2,4]? YES
        Work = [7,2,4] + [0,0,2] = [7,2,6]

All processes marked → NO DEADLOCK
Safe sequence: P0 → P2 → P1 → P3 → P4
```

### When to Run Detection?

| Frequency | Trade-off |
|-----------|-----------|
| Every resource request | Expensive but catches immediately |
| Periodically (e.g., every 5 min) | Cheaper but delayed detection |
| When CPU utilization drops below threshold | Good heuristic |

---

## 5.5 Deadlock Prevention

Break one of the four necessary conditions:

### Breaking Mutual Exclusion
```
Make resources sharable where possible.
Example: Read-only files don't need mutual exclusion.
But: Printers, mutexes inherently need exclusion → hard to break.
```

### Breaking Hold and Wait
```
Option A: Request ALL resources at once before starting
  lock(A, B, C);  // atomic multi-lock
  // do work
  unlock(A, B, C);
  
  Problem: Low resource utilization, may not know needs upfront

Option B: Release all before requesting new ones
  lock(A);
  // use A
  unlock(A);
  lock(B);
  // use B
```

### Breaking No Preemption
```
If a process holding resources can't get what it needs:
  - Release ALL its current resources
  - Re-request everything later
  
  Works for: saveable state (CPU registers, memory pages)
  Doesn't work for: printers (can't un-print), database locks
```

### Breaking Circular Wait ⭐ (Most Practical)
```
Impose a GLOBAL ORDERING on all resources.
Always acquire in increasing order.

Resource ordering: R1 < R2 < R3 < R4

Thread A:         Thread B:
lock(R1);         lock(R1);    ← Same order!
lock(R3);         lock(R2);
                  lock(R3);

NO circular wait possible.
```

### 🔥 Google Interview Insight
> **Q: How does Google's codebase prevent deadlocks in practice?**
> 
> A: Primary strategy is **lock ordering** with annotations. Google's `ABSL_GUARDED_BY()` and `ABSL_ACQUIRED_BEFORE()` annotations let the compiler statically detect lock ordering violations at compile time. Additionally:
> - Lock hierarchies (levels): never acquire a lower-level lock while holding a higher-level one
> - Prefer higher-level abstractions (channels, actors) over raw locks
> - Use lock-free data structures where appropriate
> - ThreadSanitizer (TSan) dynamically detects data races and lock-order inversions in testing

---

## 5.6 Deadlock Avoidance — Banker's Algorithm

Instead of preventing deadlock structurally, **avoid** unsafe states dynamically.

### Safe State

A state is **safe** if there exists a sequence of processes that can all finish:

```
SAFE STATE:
  "I can guarantee everyone finishes 
   no matter what they request next"

UNSAFE STATE ≠ DEADLOCK:
  "There's a possibility of deadlock,
   but it might not happen"

DEADLOCK ⊂ UNSAFE ⊂ ALL STATES:
  ┌─────────────────────────┐
  │      ALL STATES         │
  │   ┌─────────────────┐   │
  │   │  UNSAFE STATES  │   │
  │   │  ┌───────────┐  │   │
  │   │  │ DEADLOCK  │  │   │
  │   │  └───────────┘  │   │
  │   └─────────────────┘   │
  └─────────────────────────┘
```

### Banker's Algorithm

```
Data Structures:
  Available[m]  = available resources of each type
  Max[n][m]     = maximum demand of each process
  Allocation[n][m] = currently allocated to each process
  Need[n][m]    = Max - Allocation (remaining need)

Safety Check:
  1. Work = Available
     Finish[i] = false for all i
  2. Find process Pi where:
     Finish[i] == false AND Need[i] <= Work
  3. If found:
     Work = Work + Allocation[i]  // simulate Pi finishing
     Finish[i] = true
     Go to step 2
  4. If all Finish[i] == true → SAFE
     Else → UNSAFE

Resource Request (Process Pi requests Request[]):
  1. If Request > Need[i] → ERROR (exceeds max claim)
  2. If Request > Available → Pi must wait
  3. Pretend to allocate:
     Available -= Request
     Allocation[i] += Request
     Need[i] -= Request
  4. Run safety check
     If SAFE → grant request
     If UNSAFE → deny request, rollback pretend allocation
```

### Worked Example

```
5 processes, 3 resource types (A=10, B=5, C=7 total)

         Allocation    Max       Need      Available
         A  B  C    A  B  C    A  B  C    A  B  C
P0       0  1  0    7  5  3    7  4  3    3  3  2
P1       2  0  0    3  2  2    1  2  2
P2       3  0  2    9  0  2    6  0  0
P3       2  1  1    2  2  2    0  1  1
P4       0  0  2    4  3  3    4  3  1

Is this safe? Find a safe sequence:

Step 1: Work=[3,3,2]. Check P1: Need=[1,2,2] <= [3,3,2]? YES
  Work = [3,3,2] + [2,0,0] = [5,3,2]. Finish P1.

Step 2: Work=[5,3,2]. Check P3: Need=[0,1,1] <= [5,3,2]? YES
  Work = [5,3,2] + [2,1,1] = [7,4,3]. Finish P3.

Step 3: Work=[7,4,3]. Check P4: Need=[4,3,1] <= [7,4,3]? YES
  Work = [7,4,3] + [0,0,2] = [7,4,5]. Finish P4.

Step 4: Work=[7,4,5]. Check P2: Need=[6,0,0] <= [7,4,5]? YES
  Work = [7,4,5] + [3,0,2] = [10,4,7]. Finish P2.

Step 5: Work=[10,4,7]. Check P0: Need=[7,4,3] <= [10,4,7]? YES
  Work = [10,4,7] + [0,1,0] = [10,5,7]. Finish P0.

Safe sequence: P1 → P3 → P4 → P2 → P0 ✅

Now: P1 requests [1,0,2]. Should we grant it?
  1. [1,0,2] <= Need[1]=[1,2,2]? YES
  2. [1,0,2] <= Available=[3,3,2]? YES
  3. Pretend: Available=[2,3,0], Alloc[1]=[3,0,2], Need[1]=[0,2,0]
  4. Safety check with new state... (run algorithm again)
```

### Banker's Algorithm Limitations

| Limitation | Why it matters |
|-----------|---------------|
| Must know **max** resources in advance | Often unknown |
| Must have **fixed** number of processes | Processes come and go |
| **O(n² × m)** per request | Too slow for high-frequency |
| Resources must be **returnable** | Not all resources are |

> In practice, Banker's algorithm is rarely used in general-purpose OSes. It's used in specialized real-time and embedded systems.

---

## 5.7 Deadlock Recovery

If deadlock is detected, how to recover?

### Process Termination
```
Option 1: Kill ALL deadlocked processes
  + Simple
  - Expensive, loses all work

Option 2: Kill processes one at a time until cycle breaks
  + Less wasteful
  - How to choose victim?
  
Victim selection criteria:
  - Priority of the process
  - How long it has been running
  - How many resources it holds
  - How much more it needs
  - Is it interactive or batch?
```

### Resource Preemption
```
Take resources from one process, give to another.
Challenges:
  1. Selecting a victim (minimize cost)
  2. Rollback (process must restart or checkpoint)
  3. Starvation (same process picked repeatedly)
     Fix: include rollback count in victim selection
```

---

## 5.8 Livelock

Like deadlock, but processes are NOT blocked — they keep changing state without making progress.

```
Hallway analogy:
  Person A steps LEFT to avoid B
  Person B steps RIGHT to avoid A
  Both still blocking each other
  They keep moving but never pass!

Code example:
Thread A:                    Thread B:
while (true) {               while (true) {
    lock(A);                     lock(B);
    if (!trylock(B)) {           if (!trylock(A)) {
        unlock(A);               unlock(B);
        // retry!                // retry!
    }                            }
}                             }
// Both keep acquiring and releasing without progress
```

**Solution**: Add random backoff (like Ethernet's exponential backoff):
```c
if (!trylock(B)) {
    unlock(A);
    usleep(rand() % 1000);  // Random delay before retry
}
```

---

## 📝 Interview Questions — Test Yourself

### Conceptual
1. Name and explain the four necessary conditions for deadlock.
2. What is the difference between deadlock prevention and deadlock avoidance?
3. Can a system be in an unsafe state without being deadlocked?
4. What is livelock? How is it different from deadlock?
5. Why is breaking circular wait the most practical prevention method?

### Problem Solving
6. Given a resource allocation graph, determine if there's a deadlock.
7. Run the Banker's algorithm on a given state. Is it safe? What's the safe sequence?
8. Three threads each need two of three resources {A, B, C}. Design a lock ordering to prevent deadlock.

### Design
9. Design a deadlock detection system for a distributed database.
10. How would you handle deadlocks in a multi-threaded web server?

### Answers Guide

<details>
<summary>Click to reveal answers</summary>

**3.** Yes! An unsafe state means deadlock is *possible* but not certain. Only if processes request their maximum resources in the worst possible order will deadlock actually occur. In many cases, processes finish without ever requesting their maximum.

**5.** Mutual exclusion can't be broken for many resources. Hold-and-wait requires knowing all needs upfront. No-preemption works only for saveable resources. Circular wait via resource ordering is: (a) always applicable, (b) doesn't require future knowledge, (c) has no runtime overhead, (d) can be statically verified. The only cost is programming discipline.

**9.** Distributed deadlock detection is hard because no single node has complete state:
- **Centralized**: One coordinator collects wait-for info from all nodes. Simple but single point of failure.
- **Distributed**: Each node runs local detection. When an edge crosses nodes, send probe messages. If probe returns to originator → cycle → deadlock. (Chandy-Misra-Haas algorithm)
- **Timeout-based**: If a transaction waits too long, assume deadlock and abort. Simple and widely used (e.g., database transaction timeouts).

**10.** Web server strategies:
- Use lock ordering for all mutex acquisitions
- Set timeouts on lock acquisitions (`pthread_mutex_timedlock`)
- Use lock-free data structures for hot paths
- Thread sanitizer in testing
- Monitor and alert on thread stalls
- Design to minimize shared state (prefer message passing)

</details>

---

*Next: [Module 6: Memory Management →](../06-Memory-Management/README.md)*
