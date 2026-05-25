# Module 14: Worked Practice Problem Sets

Numerical problems with step-by-step solutions for every major OS topic. Practice these on paper before checking answers.

---

## Set 1: CPU Scheduling

### Problem 1.1: Round Robin (q=3)

```
Process  Arrival  Burst
P1       0        8
P2       1        4
P3       2        2
P4       3        1
```

<details>
<summary>Solution</summary>

```
Gantt Chart:
|P1 |P2 |P3|P4|P1 |P2|P1 |
0   3   6  8  9  12 13 15

Ready queue trace:
t=0:  [P1]           → run P1 (3ms)
t=3:  [P2,P3,P1]     → run P2 (3ms), P3 arrived at 2, P1 re-queued
t=6:  [P3,P4,P1,P2]  → run P3 (2ms, finishes!)
t=8:  [P4,P1,P2]     → run P4 (1ms, finishes!)
t=9:  [P1,P2]        → run P1 (3ms)
t=12: [P2,P1]        → run P2 (1ms, finishes!)
t=13: [P1]           → run P1 (2ms, finishes!)

Turnaround = completion - arrival:
P1: 15-0=15, P2: 13-1=12, P3: 8-2=6, P4: 9-3=6
Average turnaround = (15+12+6+6)/4 = 9.75

Waiting = turnaround - burst:
P1: 15-8=7, P2: 12-4=8, P3: 6-2=4, P4: 6-1=5
Average waiting = (7+8+4+5)/4 = 6.0
```
</details>

### Problem 1.2: SRTF (Preemptive SJF)

```
Process  Arrival  Burst
P1       0        7
P2       2        4
P3       4        1
P4       5        4
```

<details>
<summary>Solution</summary>

```
t=0: P1(7) only → run P1
t=2: P1(5) vs P2(4) → P2 shorter, preempt! Run P2
t=4: P1(5) vs P2(2) vs P3(1) → P3 shortest! Run P3
t=5: P1(5) vs P2(2) vs P4(4) → P2 shortest! Run P2
t=7: P1(5) vs P4(4) → P4 shorter! Run P4
t=11: P1(5) only → run P1
t=16: done

Gantt: |P1  |P2  |P3|P2 |P4   |P1     |
       0    2    4  5   7    11      16

Waiting: P1:16-7-0=9, P2:7-4-2=1, P3:5-1-4=0, P4:11-4-5=2
Average waiting = (9+1+0+2)/4 = 3.0
```
</details>

---

## Set 2: Page Replacement

### Problem 2.1: LRU vs FIFO vs OPT (3 frames)

```
Reference string: 1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5
```

<details>
<summary>Solution</summary>

**FIFO** (3 frames):
```
Ref:  1  2  3  4  1  2  5  1  2  3  4  5
F1:  [1] 1  1 [4] 4  4 [5] 5  5  5 [4] 4
F2:      [2] 2  2 [1] 1  1 [1] 1 [3] 3 [5]
F3:          [3] 3  3 [2] 2  2 [2] 2  2  2
      F  F  F  F  F  F  F        F  F  F
Page faults: 10
```

**LRU** (3 frames):
```
Ref:  1  2  3  4  1  2  5  1  2  3  4  5
F1:  [1] 1  1 [4] 4 [2] 2  2  2  2 [4] 4
F2:      [2] 2  2 [1] 1 [5] 5  5 [3] 3 [5]
F3:          [3] 3  3  3  3 [1] 1  1  1  1
      F  F  F  F  F  F  F  F     F  F  F
Page faults: 10

LRU evicts: least recently used page at time of fault.
```

**OPT** (3 frames):
```
Ref:  1  2  3  4  1  2  5  1  2  3  4  5
F1:  [1] 1  1  1  1  1  1  1  1 [3] 3  3
F2:      [2] 2  2  2  2  2  2  2  2 [4] 4
F3:          [3][4] 4  4 [5] 5  5  5  5 [5]
      F  F  F  F           F        F  F
Page faults: 7 (optimal!)
```
</details>

### Problem 2.2: Second Chance (Clock) Algorithm (4 frames)

```
Reference string: 7, 0, 1, 2, 0, 3, 0, 4
Initial: all frames empty, clock hand at frame 0
```

<details>
<summary>Solution</summary>

```
Step 1: 7 → empty frame 0. Frames: [7*] [ ] [ ] [ ]  hand→1
Step 2: 0 → empty frame 1. Frames: [7*] [0*] [ ] [ ]  hand→2
Step 3: 1 → empty frame 2. Frames: [7*] [0*] [1*] [ ]  hand→3
Step 4: 2 → empty frame 3. Frames: [7*] [0*] [1*] [2*]  hand→0
Step 5: 0 → already in frame 1! Set ref bit. Frames: [7*] [0*] [1*] [2*]
Step 6: 3 → need to evict. Hand at 0:
  Frame 0 (7*): ref=1 → clear, advance. [7] [0*] [1*] [2*] hand→1
  Frame 1 (0*): ref=1 → clear, advance. [7] [0] [1*] [2*] hand→2
  Frame 2 (1*): ref=1 → clear, advance. [7] [0] [1] [2*] hand→3
  Frame 3 (2*): ref=1 → clear, advance. [7] [0] [1] [2] hand→0
  Frame 0 (7): ref=0 → EVICT! Replace with 3.
  Frames: [3*] [0] [1] [2]  hand→1
Step 7: 0 → already in frame 1! Set ref bit. Frames: [3*] [0*] [1] [2]
Step 8: 4 → need to evict. Hand at 1:
  Frame 1 (0*): ref=1 → clear, advance. hand→2
  Frame 2 (1): ref=0 → EVICT! Replace with 4.
  Frames: [3*] [0] [4*] [2]  hand→3

(* = reference bit set)
Page faults: 6 (steps 1-4 compulsory + steps 6, 8)
```
</details>

---

## Set 3: Banker's Algorithm

### Problem 3.1: Safety Check

```
5 processes, 3 resource types. Total: A=10, B=5, C=7

         Allocation    Max       Available
         A  B  C    A  B  C    A  B  C
P0       0  1  0    7  5  3    3  3  2
P1       2  0  0    3  2  2
P2       3  0  2    9  0  2
P3       2  1  1    2  2  2
P4       0  0  2    4  3  3

a) Is the system in a safe state?
b) Can P1 request (1,0,2)?
```

<details>
<summary>Solution</summary>

**a) Calculate Need = Max - Allocation:**
```
         Need
         A  B  C
P0       7  4  3
P1       1  2  2
P2       6  0  0
P3       0  1  1
P4       4  3  1
```

**Safety algorithm (Available = [3,3,2]):**
```
1. P1: Need[1,2,2] ≤ Avail[3,3,2]? YES
   Avail = [3,3,2]+[2,0,0] = [5,3,2]

2. P3: Need[0,1,1] ≤ [5,3,2]? YES
   Avail = [5,3,2]+[2,1,1] = [7,4,3]

3. P4: Need[4,3,1] ≤ [7,4,3]? YES
   Avail = [7,4,3]+[0,0,2] = [7,4,5]

4. P0: Need[7,4,3] ≤ [7,4,5]? YES
   Avail = [7,4,5]+[0,1,0] = [7,5,5]

5. P2: Need[6,0,0] ≤ [7,5,5]? YES
   Avail = [7,5,5]+[3,0,2] = [10,5,7] ✅

Safe sequence: <P1, P3, P4, P0, P2>
```

**b) P1 requests (1,0,2):**
```
Request[1,0,2] ≤ Need[1,2,2]? YES
Request[1,0,2] ≤ Available[3,3,2]? YES

Pretend allocation:
  Avail = [3,3,2]-[1,0,2] = [2,3,0]
  Alloc[P1] = [2,0,0]+[1,0,2] = [3,0,2]
  Need[P1] = [1,2,2]-[1,0,2] = [0,2,0]

Safety check with Avail=[2,3,0]:
  P1: Need[0,2,0] ≤ [2,3,0]? YES → Avail=[5,3,2]
  P3: Need[0,1,1] ≤ [5,3,2]? YES → Avail=[7,4,3]
  P4: Need[4,3,1] ≤ [7,4,3]? YES → Avail=[7,4,5]
  P0: Need[7,4,3] ≤ [7,4,5]? YES → Avail=[7,5,5]
  P2: Need[6,0,0] ≤ [7,5,5]? YES → Avail=[10,5,7]

SAFE → Grant the request ✅
```
</details>

---

## Set 4: Address Translation

### Problem 4.1: Two-Level Page Table

```
32-bit virtual address, 4KB pages, 4-byte PTEs
Page table split: 10-bit L1, 10-bit L2, 12-bit offset

Virtual address: 0x00403004
```

<details>
<summary>Solution</summary>

```
Binary: 0000 0000 0100 0000 0011 0000 0000 0100

Split into fields:
L1 index (10 bits): 0000000001 = 1
L2 index (10 bits): 0000000011 = 3
Offset   (12 bits): 000000000100 = 4

Translation:
1. CR3 → L1 Page Directory base
2. L1[1] → gives L2 Page Table base address
3. L2[3] → gives Physical Frame Number (say, Frame 0x7A)
4. Physical Address = Frame × Page_Size + Offset
   = 0x7A × 0x1000 + 0x004 = 0x7A004

Memory accesses: 3 (L1 + L2 + data) without TLB
                 1 (data only) with TLB hit
```
</details>

### Problem 4.2: Effective Access Time

```
TLB hit ratio: 98%
TLB access: 2ns
Memory access: 100ns
Page table levels: 2
Page fault rate: 0.001 (0.1%)
Disk access: 5ms = 5,000,000ns
```

<details>
<summary>Solution</summary>

```
Case 1: TLB hit, no page fault
  Time = TLB + Memory = 2 + 100 = 102ns
  Probability = 0.98 × 0.999 = 0.97902

Case 2: TLB miss, no page fault
  Time = TLB + (2+1)×Memory = 2 + 300 = 302ns
  (2 page table accesses + 1 data access)
  Probability = 0.02 × 0.999 = 0.01998

Case 3: TLB miss + page fault
  Time = TLB + 2×Memory + Disk + Memory = 2 + 200 + 5000000 + 100
       = 5,000,302ns
  Probability = 0.02 × 0.001 = 0.00002
  (page fault only on TLB miss)

Actually, page fault can happen on TLB hit too if the page was evicted
but let's simplify:

Case 3 probability = 0.001 (page fault rate)
Adjusted:

EAT = 0.979 × 102 + 0.020 × 302 + 0.001 × 5,000,302
    = 99.86 + 6.04 + 5000.30
    = 5106.20 ns

The page fault cost dominates even at 0.1% fault rate!
Without page faults: EAT ≈ 106ns
```
</details>

---

## Set 5: Disk Scheduling

### Problem 5.1: All Algorithms

```
Disk: 200 cylinders (0-199)
Head starts at: 53
Direction: moving toward higher cylinders
Request queue: 98, 183, 37, 122, 14, 124, 65, 67
```

<details>
<summary>Solution</summary>

**FCFS:** 53→98→183→37→122→14→124→65→67
```
Movements: 45+85+146+85+108+110+59+2 = 640
```

**SSTF:** Always go to nearest request
```
53→65(12)→67(2)→37(30)→14(23)→98(84)→122(24)→124(2)→183(59) = 236
```

**SCAN (elevator, moving up):**
```
53→65→67→98→122→124→183→[199]→37→14
53→65(12)→67(2)→98(31)→122(24)→124(2)→183(59)→199(16)→37(162)→14(23)
= 331
```

**C-SCAN (moving up, jump to 0):**
```
53→65→67→98→122→124→183→[199→0]→14→37
= 12+2+31+24+2+59+16 + (199→0 jump) + 14+23 = 382 (counting jump as 199)
or without counting jump: 183
```

**C-LOOK (moving up, jump to lowest request):**
```
53→65→67→98→122→124→183→[jump to 14]→14→37
= 12+2+31+24+2+59 + 169 + 23 = 322
```

**Summary:**
| Algorithm | Total Movement |
|-----------|---------------|
| FCFS | 640 |
| SSTF | 236 |
| SCAN | 331 |
| C-LOOK | 322 |
</details>

---

## Set 6: Synchronization Tracing

### Problem 6.1: Trace the Producer-Consumer

```c
// Buffer size = 2, empty = sem(2), full = sem(0), mutex
// Producer produces: A, B, C
// Consumer consumes items

Trace the execution showing semaphore values:
```

<details>
<summary>Solution</summary>

```
Time  Action              empty  full  mutex  Buffer
──────────────────────────────────────────────────────
0     Initial             2      0     1      [ , ]
1     P: wait(empty)      1      0     1      [ , ]
2     P: lock(mutex)      1      0     0      [ , ]
3     P: produce A        1      0     0      [A, ]
4     P: unlock(mutex)    1      0     1      [A, ]
5     P: signal(full)     1      1     1      [A, ]
6     P: wait(empty)      0      1     1      [A, ]
7     P: lock(mutex)      0      1     0      [A, ]
8     P: produce B        0      1     0      [A,B]
9     P: unlock(mutex)    0      1     1      [A,B]
10    P: signal(full)     0      2     1      [A,B]
11    P: wait(empty)      ← BLOCKS! (empty=0, buffer full)
      C: wait(full)       0      1     1      [A,B]
12    C: lock(mutex)      0      1     0      [A,B]
13    C: consume A        0      1     0      [ ,B]
14    C: unlock(mutex)    0      1     1      [ ,B]
15    C: signal(empty)    1      1     1      [ ,B]
      P: UNBLOCKED!       0      1     1      [ ,B]
16    P: lock(mutex)      0      1     0      [ ,B]
17    P: produce C        0      1     0      [C,B]
...
```
</details>

---

## Set 7: Quick Calculations

### Formulas to Memorize

```
Page table size = (Virtual address space / Page size) × PTE size

TLB EAT = h(t_tlb + t_mem) + (1-h)(t_tlb + (levels+1) × t_mem)

Internal fragmentation (avg) = Page size / 2

Max file size (Unix inode, 4KB blocks, 4B pointers):
  Direct:         12 × 4KB = 48KB
  Single indirect: 1024 × 4KB = 4MB
  Double indirect: 1024² × 4KB = 4GB
  Triple indirect: 1024³ × 4KB = 4TB

Disk access time = Seek time + Rotational latency + Transfer time
Rotational latency (avg) = 60/(RPM×2) seconds

CPU utilization (multiprogramming):
  U = 1 - p^n  (p = I/O wait fraction, n = processes)

Scheduling:
  Turnaround = Completion - Arrival
  Waiting = Turnaround - Burst
  Response = First_run - Arrival

Banker's: Need = Max - Allocation
```

---

*← [Back to Main Guide](../README.md)*
