# Module 3: CPU Scheduling

## 🧠 What You'll Learn
- Why we need scheduling and what makes a good scheduler
- All major scheduling algorithms with visual traces
- Real-world schedulers: Linux CFS, MLFQ
- Multiprocessor scheduling challenges
- Real-time scheduling basics

---

## 3.1 Why Scheduling Matters

The CPU is a **shared resource**. When multiple processes are ready to run, the **scheduler** decides who goes next.

### Scheduling Criteria

| Metric | Goal | Matters For |
|--------|------|-------------|
| **CPU Utilization** | Keep CPU busy (maximize %) | System efficiency |
| **Throughput** | Max processes completed per unit time | Batch systems |
| **Turnaround Time** | Min time from submission to completion | Batch jobs |
| **Waiting Time** | Min time spent in ready queue | All systems |
| **Response Time** | Min time from request to first response | Interactive systems |

> **The fundamental tradeoff**: Optimizing for throughput (long CPU bursts) conflicts with optimizing for response time (frequent short switches).

### When Does Scheduling Happen?

```
1. Process switches from Running → Waiting    (non-preemptive)
2. Process switches from Running → Ready       (preemptive - timer)
3. Process switches from Waiting → Ready       (preemptive)
4. Process terminates                          (non-preemptive)
```

**Non-preemptive (cooperative)**: Process runs until it voluntarily gives up CPU
**Preemptive**: OS can forcibly take CPU away (via timer interrupt)

---

## 3.2 Scheduling Algorithms

### First Come, First Served (FCFS)

The simplest: process that arrives first runs first. Non-preemptive.

```
Example: Processes arrive at time 0
┌─────────┬────────────┐
│ Process │ Burst Time │
├─────────┼────────────┤
│   P1    │     24     │
│   P2    │      3     │
│   P3    │      3     │
└─────────┴────────────┘

Gantt Chart:
|─────── P1 ───────|─ P2 ─|─ P3 ─|
0                  24     27     30

Waiting times: P1=0, P2=24, P3=27
Average waiting time = (0+24+27)/3 = 17.0
```

**Convoy Effect**: Short processes stuck behind a long one. If P2 and P3 went first:

```
|P2|P3|─────── P1 ───────|
0  3  6                  30

Average waiting time = (6+0+3)/3 = 3.0  ← Much better!
```

### Shortest Job First (SJF)

Pick the process with the shortest CPU burst. **Provably optimal** for minimizing average waiting time.

```
┌─────────┬────────────┬──────────┐
│ Process │ Burst Time │ Arrival  │
├─────────┼────────────┼──────────┤
│   P1    │     6      │    0     │
│   P2    │     8      │    0     │
│   P3    │     7      │    0     │
│   P4    │     3      │    0     │
└─────────┴────────────┴──────────┘

SJF Order: P4(3) → P1(6) → P3(7) → P2(8)

|─P4─|──P1──|──P3───|───P2───|
0    3      9      16       24

Waiting: P4=0, P1=3, P3=9, P2=16
Average = (0+3+9+16)/4 = 7.0
```

**Problem**: How do you know the burst time in advance? You don't!

**Solution — Exponential Averaging**:
```
τ(n+1) = α × t(n) + (1-α) × τ(n)

Where:
  τ(n+1) = predicted next burst
  t(n)   = actual length of last burst
  τ(n)   = predicted length of last burst
  α      = weight (typically 0.5)
```

### Shortest Remaining Time First (SRTF) — Preemptive SJF

```
┌─────────┬────────────┬──────────┐
│ Process │ Burst Time │ Arrival  │
├─────────┼────────────┼──────────┤
│   P1    │     8      │    0     │
│   P2    │     4      │    1     │
│   P3    │     9      │    2     │
│   P4    │     5      │    3     │
└─────────┴────────────┴──────────┘

Timeline:
t=0: Only P1 available → run P1 (remaining=8)
t=1: P2 arrives (burst=4), P1 remaining=7 → P2 is shorter, preempt!
t=2: P3 arrives (burst=9), P2 remaining=3 → P2 still shortest
t=3: P4 arrives (burst=5), P2 remaining=2 → P2 still shortest
t=5: P2 done. P1(5), P3(9), P4(5) → P1 or P4 (tie)
t=10: P1 done. P4(5), P3(9) → P4
t=15: P4 done. P3(9) → P3
t=24: P3 done.

|P1|──P2──|───P1───|──P4──|─────P3─────|
0  1      5       10     15           24
```

### Round Robin (RR)

Each process gets a fixed **time quantum** (q). After q expires, process is preempted and goes to back of ready queue.

```
Time quantum = 4

┌─────────┬────────────┐
│ Process │ Burst Time │
├─────────┼────────────┤
│   P1    │     24     │
│   P2    │      3     │
│   P3    │      3     │
└─────────┴────────────┘

|─P1─|P2|P3|─P1─|─P1─|─P1─|─P1─|─P1─|
0    4  7 10   14   18   22   26   30

P2 and P3 finish quickly → better response time!
Average waiting = (6+4+7)/3 = 5.67
```

### The Time Quantum Dilemma

```
q too large → degenerates to FCFS (bad response time)
q too small → too many context switches (overhead dominates)

Sweet spot: q should be large enough that 80% of CPU bursts
            complete within one quantum.
            
Typical values: 10-100 ms
```

### Priority Scheduling

Each process gets a priority. Higher priority runs first.

```
┌─────────┬────────────┬──────────┐
│ Process │ Burst Time │ Priority │
├─────────┼────────────┼──────────┤
│   P1    │     10     │    3     │
│   P2    │      1     │    1     │ ← Highest
│   P3    │      2     │    4     │
│   P4    │      1     │    5     │
│   P5    │      5     │    2     │
└─────────┴────────────┴──────────┘
(Lower number = higher priority)

Order: P2(1) → P5(2) → P1(3) → P3(4) → P4(5)

|P2|──P5──|────P1────|P3─|P4|
0  1      6        16  18 19
```

**Problem — Starvation**: Low-priority processes may never run.

**Solution — Aging**: Gradually increase the priority of waiting processes.

```
Every T seconds in ready queue: priority += 1
Eventually, even the lowest priority process will get high enough to run.
```

---

## 3.3 Multilevel Feedback Queue (MLFQ)

The **most important practical scheduling algorithm**. Used as the basis for most real OS schedulers.

### Rules of MLFQ

```
Rule 1: If Priority(A) > Priority(B) → A runs
Rule 2: If Priority(A) = Priority(B) → A and B run in Round Robin
Rule 3: New jobs start at the HIGHEST priority queue
Rule 4: If a job uses its entire time slice → move DOWN one queue
Rule 5: After time period S → move ALL jobs to the highest queue (boost)
```

### Visual Representation

```
Queue 0 (Highest Priority, q=8ms):  [Interactive jobs]
   ┌──┐ ┌──┐
   │P1│ │P3│  ← Short burst jobs get served first
   └──┘ └──┘
   
Queue 1 (Medium Priority, q=16ms):
   ┌──┐
   │P5│         ← Jobs that used full quantum at Q0
   └──┘

Queue 2 (Lowest Priority, FCFS):
   ┌──┐ ┌──┐
   │P2│ │P4│   ← Long-running CPU-bound jobs
   └──┘ └──┘
```

### Why MLFQ Works

```
Interactive process (I/O bound):
  Enters at Q0 → does short burst → blocks on I/O → 
  comes back → still at Q0 → great response time! ✅

CPU-bound process:
  Enters at Q0 → uses full quantum → drops to Q1 →
  uses full quantum → drops to Q2 → runs in background ✅

Gaming the system (fake I/O):
  Process could issue fake I/O just before quantum expires →
  stays at Q0 → Solution: Track TOTAL CPU time per level,
  not just last burst ✅
```

---

## 3.4 Linux Completely Fair Scheduler (CFS)

The actual scheduler used in Linux (since 2.6.23, 2007). Replaced by EEVDF in 6.6 (2023) but CFS concepts remain fundamental.

### Key Idea: Virtual Runtime

Instead of time slices, CFS tracks **virtual runtime (vruntime)** — how much CPU time a process has used (weighted by priority).

```
vruntime = actual_runtime × (NICE_0_WEIGHT / process_weight)

- Higher priority → lower weight denominator → vruntime grows slower
- CFS always picks the process with the LOWEST vruntime
- Naturally self-balancing: if a process runs a lot, its vruntime increases,
  so other processes get picked
```

### Data Structure: Red-Black Tree

```
CFS maintains a red-black tree ordered by vruntime:

            [vruntime=50]
           /              \
    [vruntime=20]    [vruntime=80]
    /          \            \
[vruntime=10] [vruntime=30] [vruntime=90]
     ↑
   Next to run! (leftmost node = smallest vruntime)

Lookup: O(1) with cached leftmost pointer
Insert/Delete: O(log n)
```

### Nice Values and Weights

```
Nice value range: -20 (highest priority) to +19 (lowest priority)

Nice=0:  weight=1024 (baseline)
Nice=-5: weight=3121 (runs ~3x more)
Nice=+5: weight=335  (runs ~3x less)

Each nice level is roughly a 10% difference in CPU share.
```

### 🔥 Google Interview Insight
> **Q: How does CFS ensure fairness when processes have different priorities?**
> 
> A: CFS uses **weighted virtual runtime**. A high-priority process (nice=-5, weight=3121) running for 1ms accumulates `vruntime = 1ms × 1024/3121 ≈ 0.33ms`. A normal process running for 1ms accumulates `vruntime = 1ms × 1024/1024 = 1ms`. So the high-priority process can run ~3x longer before its vruntime catches up, naturally giving it ~3x the CPU share without explicit time slices.

---

## 3.5 Multiprocessor Scheduling

### Problems Unique to Multi-CPU Scheduling

```
1. CACHE AFFINITY
   Process ran on CPU0 → data is in CPU0's cache
   If rescheduled on CPU1 → cold cache → slow!
   Solution: Try to keep process on the same CPU

2. LOAD BALANCING
   CPU0: [P1, P2, P3, P4]    CPU1: []     ← Bad!
   Need to migrate processes, but this hurts cache affinity

3. SCALABILITY
   Single run queue with lock → all CPUs contend for one lock
   Per-CPU run queues → need periodic rebalancing
```

### Per-CPU Run Queues (Linux Approach)

```
CPU0 Run Queue        CPU1 Run Queue        CPU2 Run Queue
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ P1, P4, P7   │     │ P2, P5       │     │ P3, P6, P8   │
│ (RB-tree)    │     │ (RB-tree)    │     │ (RB-tree)    │
└──────────────┘     └──────────────┘     └──────────────┘

Periodic load balancing:
- Check every 1ms (busy) to 4ms (idle)
- Migration if imbalance exceeds threshold
- Consider NUMA topology (prefer nearby CPUs)
```

### NUMA-Aware Scheduling

```
NUMA (Non-Uniform Memory Access):

┌──────────────────┐      ┌──────────────────┐
│     Node 0       │      │     Node 1       │
│  ┌────┐ ┌────┐   │      │  ┌────┐ ┌────┐   │
│  │CPU0│ │CPU1│   │      │  │CPU2│ │CPU3│   │
│  └────┘ └────┘   │      │  └────┘ └────┘   │
│  ┌────────────┐  │      │  ┌────────────┐  │
│  │ Local RAM  │  │ slow │  │ Local RAM  │  │
│  │ (fast!)    │←─┼──────┼─→│ (fast!)    │  │
│  └────────────┘  │ link │  └────────────┘  │
└──────────────────┘      └──────────────────┘

Schedule process near its memory for best performance!
```

---

## 3.6 Real-Time Scheduling

For systems where missing a deadline = failure (aircraft, medical devices).

### Two Types

| Type | Consequence of Miss | Example |
|------|-------------------|---------|
| **Hard Real-Time** | Catastrophic failure | Aircraft controls, ABS brakes |
| **Soft Real-Time** | Degraded quality | Video streaming, audio playback |

### Rate-Monotonic Scheduling (RMS)

Static priority: shorter period → higher priority.

```
Task T1: Period=5, Execution=1
Task T2: Period=8, Execution=2
Task T3: Period=10, Execution=3

Priority: T1 > T2 > T3 (shorter period = higher priority)

Timeline:
|T1|T2─|T3──|T1|T3|  |T1|T2─|T1|   |T1|T2|T3─|
0  1   3    6  7  8  10 11  13 15    20 21 23  26

CPU Utilization bound: n(2^(1/n) - 1)
For n=3: 3(2^(1/3) - 1) ≈ 0.78 → schedulable if U ≤ 78%
Our U = 1/5 + 2/8 + 3/10 = 0.75 ✅
```

### Earliest Deadline First (EDF)

Dynamic priority: closest deadline → highest priority. Optimal — can schedule if total utilization ≤ 100%.

---

## 📝 Interview Questions — Test Yourself

### Conceptual
1. Compare FCFS, SJF, and Round Robin. When is each appropriate?
2. What is the convoy effect? Which algorithm suffers from it?
3. How does MLFQ prevent starvation?
4. Explain how CFS achieves fairness without explicit time slices.
5. Why is SJF optimal for minimizing average waiting time?

### Problem Solving
6. Given processes with arrival times and burst times, compute the Gantt chart and average waiting time for RR with q=4.
7. Design a scheduler for a web server that must handle both CPU-intensive requests and quick API calls.
8. A system uses MLFQ with 3 queues (q=5ms, 10ms, FCFS). Trace a CPU-bound and I/O-bound process through the system.

### Deep Dive
9. Compare CFS (Linux) with the Windows thread scheduler. What are the key differences?
10. How does the `nice` command actually affect scheduling in Linux?
11. What is the "scheduler latency" in CFS and how is it calculated?

### Answers Guide

<details>
<summary>Click to reveal answers</summary>

**1.** FCFS: Simplest, no starvation, but convoy effect. Good for batch systems with similar jobs. SJF: Optimal average waiting time, but needs burst prediction and can cause starvation. RR: Fair, good response time, but higher turnaround. Good for interactive/time-sharing systems.

**2.** Convoy effect: many short processes wait behind one long process. FCFS suffers from it. SJF, SRTF, and RR don't (they prioritize short jobs or give everyone a turn).

**3.** MLFQ uses **priority boosting**: periodically (every S seconds), all processes are moved to the highest priority queue. This ensures that even long-running processes get a chance at high priority periodically.

**4.** CFS tracks virtual runtime (vruntime) for each process. It always runs the process with the lowest vruntime. If a process runs a lot, its vruntime increases, so it gets deprioritized. This naturally distributes CPU time proportionally to weights (derived from nice values).

**5.** Proof by exchange argument: In any non-SJF schedule, you can find two adjacent jobs where the longer one runs before the shorter one. Swapping them reduces the shorter job's waiting time by the longer job's burst time, and increases the longer job's waiting time by the shorter job's burst time. Since shorter < longer, the net change is negative → total waiting time decreases. Repeat until sorted by burst time = SJF.

**11.** Scheduler latency = the target period in which every runnable process should get at least one turn. Default is 6ms (for ≤8 processes). The time slice for each process = latency / num_runnable_processes (minimum 0.75ms). With 8 processes, each gets 6/8 = 0.75ms. With 4 processes, each gets 6/4 = 1.5ms.

</details>

---

*Next: [Module 4: Process Synchronization →](../04-Synchronization/README.md)*
