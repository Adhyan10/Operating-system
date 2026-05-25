# Module 8: I/O Systems

## 🧠 What You'll Learn
- How the OS manages I/O devices
- Programmed I/O vs Interrupt-driven I/O vs DMA
- Device drivers and the I/O software stack
- Disk scheduling algorithms (SCAN, C-SCAN, etc.)
- SSD internals and how they differ from HDDs
- Blocking vs non-blocking vs asynchronous I/O

---

## 8.1 I/O Hardware Basics

### Device Controller Architecture

```
CPU ←──── System Bus ────→ Memory
              │
              │
         ┌────┴─────┐
         │ Device    │
         │Controller │
         │ ┌──────┐  │
         │ │Status │  │ ← CPU checks this
         │ │Command│  │ ← CPU writes commands here
         │ │Data   │  │ ← Data transfer register
         │ └──────┘  │
         └────┬──────┘
              │
         ┌────┴──────┐
         │  Device   │
         │ (disk,    │
         │  keyboard)│
         └───────────┘
```

### Three I/O Methods

#### 1. Programmed I/O (Polling)

```
CPU:
  while (device.status != READY) {
      // busy wait — CPU does NOTHING useful
  }
  device.data = byte;
  device.command = WRITE;
  while (device.status != DONE) {
      // busy wait again
  }

✅ Simple
❌ CPU wastes cycles polling
❌ Terrible for slow devices
```

#### 2. Interrupt-Driven I/O

```
CPU:
  device.data = byte;
  device.command = WRITE;
  // CPU goes and does OTHER work
  
  ...later...
  
  // ← INTERRUPT from device
  interrupt_handler() {
      // Device is done!
      acknowledge_interrupt();
      // Process result
  }

✅ CPU free to do other work while device processes
❌ Interrupt per byte/word → too many interrupts for high-speed devices
❌ Each interrupt has overhead (~2-5 μs)
```

#### 3. Direct Memory Access (DMA)

```
CPU tells DMA controller:
  "Transfer 4096 bytes from memory address X to disk"

DMA Controller:
  ┌──────────────────────────┐
  │ Source Address: 0x50000  │
  │ Destination: disk sector │
  │ Count: 4096 bytes        │
  │ Direction: memory→device │
  └──────────────────────────┘

Process:
  1. CPU programs DMA controller
  2. DMA transfers data block WITHOUT CPU
  3. When complete → ONE interrupt to CPU
  4. CPU handles completion

     CPU          DMA Controller       Device
      │                │                  │
      │──program DMA──→│                  │
      │                │──transfer data──→│
      │ (free to       │──transfer data──→│
      │  do other      │──transfer data──→│
      │  work!)        │──transfer data──→│
      │←──interrupt────│   (done!)        │
      │                │                  │

✅ CPU only involved at start and end
✅ Block transfer = 1 interrupt instead of 4096
✅ Essential for high-speed I/O (disk, network, GPU)
```

---

## 8.2 I/O Software Layers

```
┌────────────────────────────────────┐
│     User Process                    │  read(fd, buf, n)
├────────────────────────────────────┤
│     Library (libc)                  │  buffering, formatting
├────────────────────────────────────┤
│     System Call Interface           │  trap to kernel
├────────────────────────────────────┤
│     Device-Independent OS Layer     │  naming, caching,
│     (VFS, block layer, etc.)       │  buffering, scheduling
├────────────────────────────────────┤
│     Device Driver                   │  device-specific code
│     (loaded as kernel module)       │  translates to registers
├────────────────────────────────────┤
│     Interrupt Handler               │  handles device interrupts
├────────────────────────────────────┤
│     Hardware (Device Controller)    │  actual I/O
└────────────────────────────────────┘
```

### Device Drivers

- Each device type needs its own driver
- Runs in **kernel space** (has full hardware access)
- Linux: ~70% of kernel code is device drivers
- Interface: implements a standard set of functions (e.g., `read()`, `write()`, `ioctl()`)

---

## 8.3 Disk Scheduling (HDD)

HDD has mechanical parts → **seek time** (moving head) dominates performance.

```
HDD Anatomy:
         ┌───────────────────┐
    ────→│    Read/Write Head │
         │    (moves radially)│
         └───────┬───────────┘
                 │
        ┌────────┴────────────┐
        │  ╱╱╱╱╱ Platter ╲╲╲╲│
        │ ╱ Track 0         ╲ │  ← Each concentric ring = track
        │╱  Track 1          ╲│
        │   Track 2  ╱Sector╲ │  ← Each pie slice = sector
        │╲           ╲      ╱ │
        │ ╲                ╱  │
        │  ╲╲╲╲╲╲╲╲╲╲╲╲╲╱   │
        └─────────────────────┘
                 ↻ spinning (7200+ RPM)

Access time = Seek time + Rotational latency + Transfer time
             (~4-10ms)    (~2-4ms)              (~0.01ms/sector)
```

### Scheduling Algorithms

Given request queue: 98, 183, 37, 122, 14, 124, 65, 67
Head starts at position 53. Disk has 200 cylinders (0-199).

#### FCFS (First Come First Served)
```
53 → 98 → 183 → 37 → 122 → 14 → 124 → 65 → 67
   45   85   146   85   108   110   59    2
Total head movement = 640 cylinders
```

#### SSTF (Shortest Seek Time First)
```
53 → 65 → 67 → 37 → 14 → 98 → 122 → 124 → 183
   12    2   30   23   84    24     2    59
Total head movement = 236 cylinders (better!)
Problem: Starvation of far-away requests
```

#### SCAN (Elevator Algorithm) ⭐
```
Head moves in one direction, services all requests, then reverses.

53 → 65 → 67 → 98 → 122 → 124 → 183 → [199] → 37 → 14
   12    2   31    24     2    59    16    162   23
Direction: → → → → → → → reverse ← ←
Total head movement = 331 cylinders
```

#### C-SCAN (Circular SCAN)
```
Always scan in one direction. At end, jump back to beginning.

53 → 65 → 67 → 98 → 122 → 124 → 183 → [199→0] → 14 → 37
More uniform wait times than SCAN.
```

#### C-LOOK
```
Like C-SCAN but don't go all the way to the end — go to last request.

53 → 65 → 67 → 98 → 122 → 124 → 183 → [jump to 14] → 14 → 37
Total: better than C-SCAN (no wasted travel to disk edges)
```

### Comparison

| Algorithm | Total Movement | Starvation? | Notes |
|-----------|---------------|-------------|-------|
| FCFS | 640 | No | Simple but inefficient |
| SSTF | 236 | Yes | Greedy, good average |
| SCAN | 331 | No | Fair, like an elevator |
| C-SCAN | ~330 | No | More uniform wait times |
| C-LOOK | ~300 | No | **Used in practice (Linux)** |

---

## 8.4 SSD Internals

SSDs have fundamentally different characteristics from HDDs:

```
SSD Architecture:
┌────────────────────────────────────┐
│           SSD Controller           │
│  ┌──────────┐  ┌───────────────┐  │
│  │ FTL      │  │ Wear Leveling │  │  FTL = Flash Translation Layer
│  │ (logical │  │ (spread writes│  │
│  │  → phys) │  │  evenly)      │  │
│  └──────────┘  └───────────────┘  │
├────────────────────────────────────┤
│  ┌──────┐ ┌──────┐ ┌──────┐      │
│  │NAND  │ │NAND  │ │NAND  │ ...  │  Multiple NAND chips
│  │Chip 0│ │Chip 1│ │Chip 2│      │  (parallel for speed)
│  └──────┘ └──────┘ └──────┘      │
└────────────────────────────────────┘
```

### HDD vs SSD Comparison

| Aspect | HDD | SSD |
|--------|-----|-----|
| **Random read** | ~10ms | ~0.1ms (100x faster) |
| **Sequential read** | ~100 MB/s | ~500-7000 MB/s |
| **Random write** | ~10ms | ~0.1ms |
| **Seek time** | 4-10ms | 0ms (no moving parts) |
| **Power** | ~6-8W | ~2-3W |
| **Durability** | Mechanical failure | Limited write cycles |
| **Cost per GB** | ~$0.03 | ~$0.10-0.20 |

### SSD Write Amplification

```
SSD writes in PAGES (4KB) but erases in BLOCKS (256KB).

To overwrite 1 page in a block:
1. Read entire block (64 pages) into buffer
2. Modify the 1 page
3. Erase entire block
4. Write back all 64 pages

Write amplification factor = data written / data requested
Can be 10-60x in worst case!

Mitigation:
- TRIM command: OS tells SSD which blocks are free
- Garbage collection: background process consolidates used pages
- Over-provisioning: extra hidden capacity for GC
```

### 🔥 Google Interview Insight
> **Q: Does disk scheduling matter for SSDs?**
> 
> A: Traditional algorithms like SCAN are unnecessary since SSDs have no seek time. However, I/O scheduling still matters:
> - **Merging**: Combine adjacent I/O requests to reduce command overhead
> - **Prioritization**: Interactive I/O over batch I/O
> - **Queue depth**: SSDs benefit from deep queues (NVMe supports 64K queues × 64K entries)
> - **Linux uses `mq-deadline` or `none` scheduler** for SSDs instead of the `bfq` used for HDDs

---

## 8.5 Blocking, Non-blocking, and Asynchronous I/O

### Three I/O Models

```
Blocking I/O:
  Thread:  ──call──│ wait... wait... wait... │──return──→
                   └────────────────────────┘
                        blocked (sleeping)
  Thread can do NOTHING while waiting.

Non-blocking I/O:
  Thread:  ──call──│return immediately│──poll──│return│──poll──│ready!│
                   (EAGAIN/EWOULDBLOCK)        (not ready)     (data!)
  Thread keeps checking. Wastes CPU but stays responsive.

Asynchronous I/O (AIO):
  Thread:  ──submit──│return immediately│──do other work──│callback!│
                     (I/O in background)                   (data ready)
  Best of both worlds: no blocking, no polling.
```

### I/O Multiplexing: select / poll / epoll

For handling many connections (e.g., web servers):

| System Call | Scalability | How it works |
|-------------|-------------|-------------|
| `select()` | O(n), max 1024 fds | Scan all file descriptors every call |
| `poll()` | O(n), no fd limit | Same as select but no limit |
| `epoll()` | O(1) per event | Kernel tracks changes, only returns ready fds |
| `io_uring` | O(1), zero-copy | Shared ring buffer, batch submission, newest |

### epoll — How Modern Servers Work

```c
int epfd = epoll_create1(0);

struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = listen_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

struct epoll_event events[MAX_EVENTS];
while (1) {
    int n = epoll_wait(epfd, events, MAX_EVENTS, -1);
    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == listen_fd) {
            // Accept new connection
        } else {
            // Handle data on existing connection
        }
    }
}
```

```
1000 connections, but only 3 have data:

select/poll: Scan all 1000 → find 3 ready → O(1000)
epoll:       Kernel tells you exactly which 3 → O(3)

For C10K+ servers, epoll is essential.
```

---

## 📝 Interview Questions — Test Yourself

### Conceptual
1. Compare polling, interrupt-driven, and DMA-based I/O.
2. What is the I/O software stack? Why is it layered?
3. Why does disk scheduling matter for HDDs but not SSDs?
4. Compare blocking, non-blocking, and asynchronous I/O.
5. What is write amplification in SSDs?

### Problem Solving
6. Given a disk request queue and starting head position, compute total head movement for FCFS, SSTF, SCAN, and C-LOOK.
7. Design an I/O scheduler for a mixed HDD+SSD system.

### Design
8. How does `io_uring` improve on `epoll`? What problems does it solve?
9. Design a buffer cache for a file system. What eviction policy would you use?
10. How does a network card (NIC) process incoming packets? Trace from wire to user space.

### Answers Guide

<details>
<summary>Click to reveal answers</summary>

**1.** Polling: CPU repeatedly checks device status — simple but wastes CPU. Interrupt-driven: device interrupts CPU when ready — CPU free between, but one interrupt per transfer unit. DMA: device controller transfers entire block, interrupts once — CPU free during transfer, best for bulk I/O.

**8.** `io_uring` improvements over `epoll`:
- **Unified interface**: handles files, sockets, timers (epoll is network-only)
- **Batched submissions**: submit many I/O requests at once via shared ring buffer
- **Zero syscall path**: submission and completion via shared memory (no syscall per operation)
- **Linked operations**: chain dependent I/O operations
- **Fixed buffers**: pre-register buffers to avoid per-I/O mapping
- Result: 2-10x better performance for I/O-heavy workloads

**10.** Network packet from wire to user space:
1. **NIC receives packet** → stores in DMA ring buffer in kernel memory
2. **NIC raises interrupt** → kernel interrupt handler runs
3. **NAPI polling** (Linux): switch to polling mode if high packet rate
4. **Network stack**: Ethernet → IP → TCP headers processed
5. **Socket buffer**: data placed in socket's receive queue
6. **Wake up process**: if blocked in `recv()` or `epoll_wait()`
7. **Copy to user space**: `recv()` copies from kernel buffer to user buffer
   (or `recvmsg()` with `MSG_ZEROCOPY` to avoid copy)

</details>

---

*Next: [Module 9: Virtualization & Containers →](../09-Virtualization/README.md)*
