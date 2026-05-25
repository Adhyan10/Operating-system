# Module 6: Memory Management

## 🧠 What You'll Learn
- Physical vs virtual memory and why virtual memory exists
- Address translation: base/limit → paging → multi-level paging
- Page tables, TLBs, and their performance impact
- Page replacement algorithms (FIFO, LRU, Clock, etc.)
- Thrashing and working set model
- Memory-mapped files, demand paging, copy-on-write

> **This module is CRITICAL for Google interviews.** Memory management is foundational to systems programming, performance optimization, and system design.

---

## 6.1 Why Virtual Memory?

### The Problems Without Virtual Memory

```
Problem 1: Protection
  Process A and B both in physical memory — nothing stops A
  from reading/writing B's data!

Problem 2: Address Space
  Every process must know exactly where it's loaded.
  Can't compile with fixed addresses — what if that address is taken?

Problem 3: Size
  Process needs more memory than physically available.
  What now?

Solution: VIRTUAL MEMORY
  Each process gets its own private address space.
  OS + hardware translate virtual → physical transparently.
```

### The Big Picture

```
Process A sees:              Physical Memory:        Process B sees:
┌──────────────┐            ┌──────────────┐        ┌──────────────┐
│ 0x0000: code │            │ Frame 0: A's │        │ 0x0000: code │
│ 0x1000: data │──mapping──→│ Frame 1: B's │←──────→│ 0x1000: data │
│ 0x2000: heap │            │ Frame 2: A's │        │ 0x2000: heap │
│ ...          │            │ Frame 3: free│        │ ...          │
│ 0xF000: stack│            │ Frame 4: B's │        │ 0xF000: stack│
└──────────────┘            │ Frame 5: OS  │        └──────────────┘
                            └──────────────┘
Both think they own                                
addresses 0x0000-0xFFFF!     
```

---

## 6.2 Address Translation Mechanisms

### Base and Limit Registers (Simplest)

```
Virtual Address:  offset
Physical Address: base + offset

┌────────────┬────────────┐
│   base     │   limit    │
│   10000    │   5000     │
└────────────┴────────────┘

Valid range: [10000, 15000)
If offset >= limit → TRAP (segfault)

Limitations:
- No sharing between processes
- External fragmentation
- Process must be contiguous in physical memory
```

### Segmentation

```
Each process has multiple segments (code, data, stack, heap).
Each segment has its own base + limit.

Segment Table:
┌─────────┬───────┬───────┐
│ Segment │ Base  │ Limit │
├─────────┼───────┼───────┤
│ Code    │ 1000  │ 2000  │
│ Data    │ 5000  │ 500   │
│ Stack   │ 8000  │ 1000  │
│ Heap    │ 3000  │ 3000  │
└─────────┴───────┴───────┘

Virtual address = [segment number : offset]

Pros: Logical grouping, sharing (share code segment)
Cons: Variable-size → EXTERNAL FRAGMENTATION
```

---

## 6.3 Paging — The Foundation of Modern Memory Management

Divide physical memory into fixed-size **frames** and virtual memory into **pages** (same size, typically 4KB).

```
Virtual Address Space          Physical Memory (RAM)
┌────────────────┐            ┌────────────────┐
│ Page 0         │──────────→ │ Frame 3        │
├────────────────┤            ├────────────────┤
│ Page 1         │──────────→ │ Frame 7        │
├────────────────┤            ├────────────────┤
│ Page 2         │──────┐     │ Frame 1        │
├────────────────┤      │     ├────────────────┤
│ Page 3         │──┐   │     │ Frame 5        │←── Page 3
├────────────────┤  │   │     ├────────────────┤
│ ...            │  │   └───→ │ Frame 2        │←── Page 2
└────────────────┘  │         ├────────────────┤
                    └────────→│ Frame 5        │
                              └────────────────┘

No external fragmentation! (All chunks are same size)
Pages can be anywhere in physical memory (non-contiguous)
Only internal fragmentation (last page partially filled)
```

### Address Translation with Paging

```
Virtual Address (32-bit, 4KB pages):
┌──────────────────────┬────────────────┐
│  Page Number (20b)   │  Offset (12b)  │
└──────────────────────┴────────────────┘
        │                      │
        │ (look up in          │ (used directly)
        │  page table)         │
        ▼                      │
┌──────────────────────┐       │
│    Page Table        │       │
│  ┌──────┬──────┐     │       │
│  │PN 0  │ FN 3 │     │       │
│  │PN 1  │ FN 7 │     │       │
│  │PN 2  │ FN 2 │     │       │
│  │PN 3  │ FN 5 │     │       │
│  └──────┴──────┘     │       │
└──────────────────────┘       │
        │                      │
        ▼                      ▼
┌──────────────────────┬────────────────┐
│  Frame Number (20b)  │  Offset (12b)  │
└──────────────────────┴────────────────┘
Physical Address
```

### Page Table Entry (PTE)

```
┌───┬───┬───┬───┬───┬───────────────────┐
│ V │ R │ M │ X │ P │  Frame Number     │
└───┴───┴───┴───┴───┴───────────────────┘

V = Valid bit (is this page in memory?)
R = Referenced bit (has this page been accessed?)
M = Modified/Dirty bit (has this page been written?)
X = Execute permission
P = Protection bits (read/write/execute, user/kernel)
Frame Number = physical frame where this page lives
```

### 🔥 Google Interview Insight
> **Q: A 64-bit system with 4KB pages — how big is the page table?**
> 
> A: 64-bit address space = 2^64 bytes. Page size = 4KB = 2^12.  
> Number of pages = 2^64 / 2^12 = 2^52 pages.  
> Each PTE = 8 bytes → Page table = 2^52 × 8 = 2^55 bytes = **32 petabytes!**  
> 
> This is why we need **multi-level page tables** — most of those pages are unused, so we don't allocate PTEs for them.

---

## 6.4 Multi-Level Page Tables

### Two-Level Page Table (32-bit example)

```
Virtual Address (32-bit, 4KB pages):
┌──────────┬──────────┬──────────────┐
│ L1 (10b) │ L2 (10b) │ Offset (12b) │
└──────────┴──────────┴──────────────┘
     │           │
     │           │
     ▼           ▼
┌─────────┐  ┌─────────┐
│ L1 Page │  │ L2 Page │
│ Directory│→│  Table  │──→ Frame Number
│         │  │         │
│ 1024    │  │ 1024    │
│ entries │  │ entries │
└─────────┘  └─────────┘

Advantage: Unused L2 tables don't need to exist!
  - If process uses only 1MB of 4GB space:
  - Only ~1 L2 table needed (instead of 1024)
  - Savings: ~4MB → ~8KB
```

### Four-Level Page Table (x86-64, Linux)

```
48-bit virtual address (canonical form):
┌──────┬──────┬──────┬──────┬──────────────┐
│PML4  │ PDP  │  PD  │  PT  │  Offset      │
│(9b)  │(9b)  │(9b)  │(9b)  │  (12b)       │
└──┬───┴──┬───┴──┬───┴──┬───┴──────────────┘
   │      │      │      │
   │      │      │      └──→ Page Table (512 entries)
   │      │      └──→ Page Directory (512 entries)
   │      └──→ Page Directory Pointer Table (512 entries)
   └──→ PML4 Table (512 entries, base in CR3 register)
   
Each level: 512 entries × 8 bytes = 4KB = one page
Total: 4 memory accesses per translation (without TLB)!
```

### Page Walk (x86-64)

```
CR3 register → PML4 Table base address
   │
   ▼
PML4[PML4_index] → PDP Table base address      Memory access 1
   │
   ▼
PDP[PDP_index] → PD Table base address          Memory access 2
   │
   ▼
PD[PD_index] → PT Table base address            Memory access 3
   │
   ▼
PT[PT_index] → Physical Frame Number            Memory access 4
   │
   + Offset → Physical Address                  Memory access 5 (data!)

5 memory accesses for 1 data access! → Need TLB
```

---

## 6.5 Translation Lookaside Buffer (TLB)

The TLB is a **hardware cache** for page table entries. Without it, paging would be impossibly slow.

```
┌──────────────────────────────────────────────────────────┐
│                         CPU                              │
│  ┌──────────────┐                                        │
│  │  TLB         │  Typical: 64-1024 entries             │
│  │  ┌────┬─────┐│  Fully associative or set-associative │
│  │  │VPN │ PFN ││  Access time: ~1 ns (1 cycle)        │
│  │  │ 0  │  3  ││                                       │
│  │  │ 5  │  7  ││                                       │
│  │  │ 2  │  1  ││                                       │
│  │  └────┴─────┘│                                       │
│  └──────────────┘                                        │
└──────────────────────────────────────────────────────────┘
```

### TLB Lookup Flow

```
Virtual Address
      │
      ▼
   ┌─────────┐
   │   TLB   │ ──Hit (99%+)──→ Frame Number → Physical Address
   │  Lookup  │                  (~1 cycle)
   └────┬────┘
        │
      Miss (~1%)
        │
        ▼
   ┌──────────┐
   │Page Table │ → 4 memory accesses (page walk)
   │   Walk    │ → Update TLB with new mapping
   └──────────┘
```

### Effective Access Time (EAT)

```
Given:
  TLB hit ratio = h (typically 0.99+)
  TLB access time = t_tlb (1 ns)
  Memory access time = t_mem (100 ns)
  Page table levels = L (4 for x86-64)

EAT = h × (t_tlb + t_mem) + (1-h) × (t_tlb + (L+1) × t_mem)

Example with h=0.99, t_tlb=1ns, t_mem=100ns, L=4:
  Hit:  1 + 100 = 101 ns
  Miss: 1 + 5 × 100 = 501 ns
  EAT = 0.99 × 101 + 0.01 × 501 = 99.99 + 5.01 = 105 ns
  
  Without TLB: 501 ns every time!
  With TLB: 105 ns → ~5x speedup even with 4-level tables
```

### TLB and Context Switches

When switching processes, TLB entries become invalid (different address space):

| Strategy | How | Cost |
|----------|-----|------|
| **Flush TLB** | Invalidate all entries | Simple but expensive (cold TLB) |
| **ASID** (Address Space ID) | Tag each entry with process ID | No flush needed, TLB can hold entries from multiple processes |

> Modern x86 uses **PCID** (Process Context ID) — equivalent to ASID. Linux enabled it in kernel 4.14.

---

## 6.6 Demand Paging

**Don't load pages until they're needed.** (Lazy loading)

```
When process starts:
  - Create page table with all entries marked INVALID
  - No pages are in memory

When process accesses a page:
  1. Check page table → Valid bit = 0 → PAGE FAULT
  2. Trap to kernel
  3. Find the page on disk (swap space or file)
  4. Find a free frame (or evict one)
  5. Load page from disk into frame
  6. Update page table (valid=1, frame=X)
  7. Restart the instruction that faulted

This is "demand" paging: pages loaded on demand.
```

### Page Fault Handling Flow

```
┌──────────────┐
│ CPU accesses  │
│ virtual addr  │
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────────────────────┐
│ TLB miss?    │─Yes→│ Page walk: check page table   │
└──────┬───────┘     └──────────┬───────────────────┘
       │No                      │
       │                        │
       ▼                        ▼
  Use TLB entry         ┌─────────────┐
                        │ Valid bit=1?│─Yes→ Update TLB, access data
                        └──────┬──────┘
                               │ No (PAGE FAULT)
                               ▼
                        ┌─────────────┐
                        │ Trap to OS  │
                        └──────┬──────┘
                               │
                               ▼
                        ┌─────────────────────┐
                        │ Page on disk?        │
                        │ Is it a valid access?│
                        └──────┬──────────────┘
                               │
                          ┌────┴────┐
                         Yes       No
                          │         │
                          ▼         ▼
                   Load from    SEGFAULT
                   disk to      (kill process)
                   free frame
                          │
                          ▼
                   Update page table
                   Restart instruction
```

### Page Fault Cost

```
Page fault = ~1-10 MILLISECONDS (disk I/O)
Normal access = ~100 NANOSECONDS

That's 10,000x - 100,000x slower!

Even with SSD: ~100 microseconds = 1000x slower

For good performance, page fault rate must be VERY low:
  If p = page fault rate:
  EAT = (1-p) × 100ns + p × 10ms
  
  For EAT < 110ns (10% overhead):
  (1-p) × 100 + p × 10,000,000 < 110
  p < 1/1,000,000
  
  Less than 1 fault per million accesses!
```

---

## 6.7 Page Replacement Algorithms

When memory is full and a page fault occurs, which page do we **evict**?

### Optimal (OPT / Belady's)

Replace the page that won't be used for the **longest time**. Not implementable (requires future knowledge) but used as a benchmark.

```
Reference string: 7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2
Frames = 3

 7  0  1  2  0  3  0  4  2  3  0  3  2
┌─┐┌─┐┌─┐┌─┐   ┌─┐   ┌─┐         ┌─┐
│7││0││1││2│   │3│   │4│         │0│
│ ││7││0││0│   │0│   │0│         │2│
│ ││ ││7││1│   │1│   │3│         │3│
└─┘└─┘└─┘└─┘   └─┘   └─┘         └─┘
 F  F  F  F     F     F           F

Page faults: 9 (optimal minimum)
```

### FIFO (First In, First Out)

Replace the **oldest** page.

```
Reference string: 7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2
Frames = 3

 7  0  1  2  0  3  0  4  2  3  0  3  2
┌─┐┌─┐┌─┐┌─┐   ┌─┐┌─┐┌─┐┌─┐┌─┐┌─┐
│7││0││1││2│   │3││0││4││2││3││0│
│ ││7││0││1│   │2││3││0││4││2││3│
│ ││ ││7││0│   │1││2││3││0││4││2│
└─┘└─┘└─┘└─┘   └─┘└─┘└─┘└─┘└─┘└─┘
 F  F  F  F     F  F  F  F  F  F

Page faults: 15 (worse than OPT)
```

**Belady's Anomaly**: With FIFO, more frames can mean MORE page faults! (Not intuitive)

### LRU (Least Recently Used) ⭐

Replace the page that hasn't been used for the **longest time**. Approximation of OPT using past behavior.

```
Reference string: 7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2
Frames = 3

 7  0  1  2  0  3  0  4  2  3  0  3  2
┌─┐┌─┐┌─┐┌─┐   ┌─┐   ┌─┐┌─┐      ┌─┐
│7││0││1││2│   │3│   │4││2│      │0│
│ ││7││0││0│   │0│   │0││0│      │3│
│ ││ ││7││1│   │2│   │3││3│      │2│
└─┘└─┘└─┘└─┘   └─┘   └─┘└─┘      └─┘
 F  F  F  F     F     F  F        F

Page faults: 12 (better than FIFO, close to OPT)
```

**LRU Implementation Options**:

| Method | How | Cost |
|--------|-----|------|
| **Counter** | Each PTE has a timestamp. Replace lowest timestamp. | O(n) scan to find min |
| **Stack** | Doubly linked list. Move to top on access. Bottom = LRU. | O(1) access, but expensive to maintain |
| **Neither** is practical at hardware speed! → Need approximations |

### Clock Algorithm (Second-Chance) — LRU Approximation

```
Frames arranged in a circular buffer with a "clock hand" pointer.
Each frame has a "reference bit" (set by hardware on access).

             ┌───┐
         ┌──→│ 1 │──┐    (1 = recently used)
         │   └───┘   │
       ┌───┐       ┌───┐
       │ 0 │       │ 1 │
       └───┘       └───┘
         │   ┌───┐   │
         └───│ 0 │←──┘
             └─▲─┘
               │
           clock hand

Algorithm:
  1. Check page at hand position
  2. If reference bit = 0 → EVICT this page
  3. If reference bit = 1 → set to 0, advance hand, go to step 1

Intuition: Pages get a "second chance" if recently used.
Worst case: goes around entire circle (all ref bits = 1) → degenerates to FIFO
```

### Enhanced Clock (NRU — Not Recently Used)

```
Use TWO bits: (Referenced, Modified)

Class 0: (0,0) — not referenced, not modified   ← Best to replace
Class 1: (0,1) — not referenced, modified
Class 2: (1,0) — referenced, not modified
Class 3: (1,1) — referenced, modified            ← Worst to replace

Replace from lowest class first.
Prefer unmodified pages (no disk write needed on eviction).
```

### 🔥 Google Interview Insight
> **Q: Implement an LRU cache with O(1) get and put.**
> 
> This is one of the most frequently asked Google interview questions!
> 
> Solution: **HashMap + Doubly Linked List**

```
┌───────────────────────────────────────────┐
│                HashMap                     │
│  key1 → Node1                             │
│  key2 → Node2                             │
│  key3 → Node3                             │
└───────────────────────────────────────────┘

┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
│ HEAD │←──→│Node3 │←──→│Node1 │←──→│ TAIL │
│(dummy)│   │(MRU) │    │      │    │(dummy)│
└──────┘    └──────┘    └──────┘    └──────┘
                         (LRU ↑)

get(key): HashMap lookup O(1) → move node to front O(1)
put(key, val): 
  - If exists: update, move to front O(1)
  - If new: add to front, if full → remove from tail (LRU) O(1)
```

---

## 6.8 Thrashing

When a process spends more time **paging** than **executing**. The system grinds to a halt.

```
CPU Utilization
     ▲
100% │     ╱╲
     │    ╱  ╲
     │   ╱    ╲
     │  ╱      ╲
     │ ╱        ╲
     │╱          ╲
     └────────────────→ Number of Processes
     
     ↑ Sweet spot   ↑ Thrashing begins!
     
As more processes are added:
1. CPU utilization increases (good!)
2. At some point, combined memory needs > physical memory
3. Excessive page faults → I/O bound → CPU idle → OS adds more processes
4. Even MORE page faults → death spiral!
```

### Working Set Model

The **working set** W(t, Δ) of a process at time t is the set of pages referenced in the last Δ time units.

```
If Σ(working set sizes) > available frames → thrashing!

Solution: Only run processes whose working sets fit in memory.
If they don't fit → suspend some processes (swap out).

Working set tracking:
- Approximate with reference bits
- Timer interrupt periodically clears reference bits
- Referenced bits that are still set → in working set
```

### Page Fault Frequency (PFF)

```
Page Fault Rate
     ▲
     │  ╲
     │   ╲   
     │    ╲──── upper threshold → give more frames
     │     ╲
     │      ────── acceptable range
     │         ╲
     │          ── lower threshold → take frames away
     │             ╲
     └────────────────→ Number of Frames
```

---

## 6.9 Memory-Mapped Files

Instead of `read()` / `write()`, map a file directly into the process's address space.

```c
int fd = open("data.bin", O_RDWR);
char *data = mmap(NULL, file_size, PROT_READ | PROT_WRITE, 
                  MAP_SHARED, fd, 0);

// Now access file like regular memory:
data[0] = 'H';         // Writes to file!
char c = data[100];    // Reads from file!

munmap(data, file_size);
close(fd);
```

```
Process Address Space:          Physical Memory:         Disk:
┌───────────────┐              ┌──────────────┐         ┌──────┐
│ mmap'd region │─── pages ──→ │ File pages   │←─ sync ─│ file │
│ 0x7f00000000  │              │ (cached)     │         │ data │
│ ...           │              └──────────────┘         └──────┘
└───────────────┘
   Reads/writes here trigger page faults that load from disk.
   Dirty pages are written back to disk by OS.
```

### Advantages of mmap

| Advantage | Why |
|-----------|-----|
| **Zero-copy** | No kernel↔user buffer copying |
| **Lazy loading** | Only pages actually accessed are loaded |
| **Shared mapping** | Multiple processes can share same physical pages |
| **Simplicity** | Treat file as array, no read/write/seek calls |

---

## 6.10 Huge Pages

Normal pages: 4KB. For large memory applications, huge page tables.

```
                    4KB Pages                    2MB Huge Pages
Mapping 1GB:       262,144 PTEs               512 PTEs
TLB coverage       4KB × 64 TLB entries        2MB × 64 TLB entries
(64-entry TLB):    = 256KB                     = 128MB

Huge pages = 500x better TLB coverage!
```

Used extensively at Google for large in-memory databases and caches.

---

## 📝 Interview Questions — Test Yourself

### Conceptual
1. Explain the full virtual-to-physical address translation process on x86-64.
2. What is a TLB? Why is it critical for performance?
3. Compare FIFO, LRU, and Clock page replacement algorithms.
4. What is thrashing? How do you detect and resolve it?
5. What is Belady's anomaly? Which algorithms suffer from it?

### Problem Solving
6. **[LeetCode 146]** Implement LRU Cache with O(1) get/put.
7. Given a reference string and N frames, compute page faults for FIFO, LRU, and OPT.
8. A 32-bit system with 4KB pages and 2-level page table: How many bits for L1, L2, and offset?

### Design
9. Design a memory allocator (like malloc). What data structures would you use?
10. How would you implement shared libraries using virtual memory?
11. Explain how memory-mapped I/O differs from port-mapped I/O.

### Answers Guide

<details>
<summary>Click to reveal answers</summary>

**5.** Belady's anomaly: More frames leading to MORE page faults. FIFO suffers from it. LRU, OPT, and all "stack algorithms" are immune. Stack algorithms: the set of pages in memory with n frames is always a subset of pages with n+1 frames.

**8.** 32-bit address, 4KB pages: Offset = log2(4096) = 12 bits. Remaining = 32 - 12 = 20 bits for page number. For 2-level: split 20 bits → 10 bits L1 + 10 bits L2. Each level has 2^10 = 1024 entries. L1 table: 1024 × 4 bytes = 4KB = 1 page ✅.

**9.** malloc design:
- **Free list**: linked list of free blocks, sorted by address or size
- **Strategies**: First fit, best fit, worst fit, buddy system
- **Metadata**: Block header with size and used/free bit
- **Splitting**: Large free block → split into allocated + remaining free
- **Coalescing**: Merge adjacent free blocks to reduce fragmentation
- **Binning**: slab allocator (size classes: 8, 16, 32, ... bytes)
- Google's **TCMalloc**: thread-local caches for small allocations, central heap for large. Dramatically reduces lock contention.

**10.** Shared libraries with VM:
- Library code pages mapped into each process's address space (shared mapping)
- All processes share the SAME physical frames for read-only code
- Each process has private data pages (COW)
- **Position Independent Code (PIC)**: code uses relative addresses, works at any virtual address
- **PLT/GOT**: Procedure Linkage Table / Global Offset Table for lazy binding of function calls

</details>

---

*Next: [Module 7: File Systems →](../07-File-Systems/README.md)*
