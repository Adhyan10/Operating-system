# Module 2: Processes and Threads

## 🧠 What You'll Learn
- Process lifecycle and state transitions
- Process Control Block (PCB) — what the OS tracks per process
- `fork()`, `exec()`, `wait()` — the Unix process model
- Threads vs Processes — when to use each
- User-level vs Kernel-level threads
- Context switching — what really happens

---

## 2.1 What is a Process?

A **process** is a **program in execution**. A program is passive (file on disk); a process is active (loaded in memory, running).

### Process Memory Layout

```
High Address
┌──────────────────┐ 0xFFFFFFFF...
│      Stack       │ ← Local variables, function calls
│   ↓ grows down   │   (each function call = new stack frame)
│                  │
├──────────────────┤
│                  │
│   (free space)   │ ← Stack and heap grow toward each other
│                  │
├──────────────────┤
│   ↑ grows up     │
│       Heap       │ ← Dynamic allocation (malloc, new)
├──────────────────┤
│       BSS        │ ← Uninitialized global variables (zeroed)
├──────────────────┤
│      Data        │ ← Initialized global/static variables
├──────────────────┤
│      Text        │ ← Machine code (read-only, sharable)
└──────────────────┘ 0x00000000
Low Address
```

### 🔥 Google Interview Insight
> **Q: What happens if stack and heap collide?**
> 
> A: Stack overflow or heap corruption. In practice, modern OSes use **virtual memory** with **guard pages** — a special unmapped page between stack and heap. Accessing it triggers a **page fault** (segfault), catching the overflow before corruption.

---

## 2.2 Process States

A process goes through several states during its lifetime:

```
                    ┌──────────────────────────────────┐
                    │                                  │
                    ▼                                  │
    ┌─────┐    ┌────────┐    ┌─────────┐    ┌──────────┤
    │ New │───→│ Ready  │───→│ Running │───→│Terminated│
    └─────┘    └────────┘    └─────────┘    └──────────┘
                  ▲               │
                  │               │ I/O or event wait
                  │               ▼
                  │          ┌─────────┐
                  └──────────│ Waiting │
                   I/O done  │(Blocked)│
                             └─────────┘
```

| State | Description |
|-------|-------------|
| **New** | Process is being created |
| **Ready** | Waiting to be assigned to a CPU |
| **Running** | Instructions are being executed |
| **Waiting** | Waiting for I/O or event |
| **Terminated** | Process has finished execution |

### Key Transitions

| Transition | Cause |
|-----------|-------|
| Ready → Running | **Scheduler dispatches** the process |
| Running → Ready | **Timer interrupt** (preemption) or `yield()` |
| Running → Waiting | Process issues **I/O request** or `wait()` |
| Waiting → Ready | **I/O completes** or event occurs |
| Running → Terminated | `exit()` or fatal signal |

---

## 2.3 Process Control Block (PCB)

The OS maintains a **PCB** (also called task_struct in Linux) for every process. It's the OS's complete knowledge about a process.

```
┌──────────────────────────────────────┐
│         Process Control Block        │
├──────────────────────────────────────┤
│  PID: 1234                           │
│  State: Running                      │
│  Program Counter: 0x7fff4a2b         │
│  CPU Registers: {rax, rbx, rsp, ...} │
│  Scheduling Info:                    │
│    Priority: 20                      │
│    CPU time used: 150ms              │
│  Memory Info:                        │
│    Page table base: 0x1000           │
│    Code segment: 0x400000-0x401000   │
│    Stack pointer: 0x7fffffffe000     │
│  I/O Info:                           │
│    Open files: [0:stdin, 1:stdout,   │
│                 2:stderr, 3:log.txt] │
│  Accounting:                         │
│    User time: 2.3s                   │
│    System time: 0.8s                 │
│  Parent PID: 1100                    │
│  Child PIDs: [1235, 1236]            │
└──────────────────────────────────────┘
```

In Linux, this is the `task_struct` — a massive struct (~6KB) defined in `include/linux/sched.h`.

---

## 2.4 Process Creation — The Unix Model

Unix/Linux uses `fork()` + `exec()` — a two-step model:

### fork()

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    printf("Parent PID: %d\n", getpid());
    
    pid_t pid = fork();  // Creates a copy of the current process
    
    if (pid < 0) {
        // Fork failed
        perror("fork failed");
        return 1;
    } else if (pid == 0) {
        // CHILD process
        printf("Child PID: %d, Parent PID: %d\n", getpid(), getppid());
        // Child has a COPY of parent's memory (Copy-on-Write)
    } else {
        // PARENT process (pid = child's PID)
        printf("Parent: created child %d\n", pid);
        wait(NULL);  // Wait for child to finish
        printf("Parent: child has terminated\n");
    }
    return 0;
}
```

### What fork() actually does:

```
Before fork():
┌──────────────────┐
│  Process (PID 100)│
│  x = 42          │
│  PC = 0x400100   │
└──────────────────┘

After fork():
┌──────────────────┐    ┌──────────────────┐
│  Parent (PID 100)│    │  Child (PID 101) │
│  x = 42          │    │  x = 42          │  ← Same values
│  fork returns 101│    │  fork returns 0  │  ← Different return!
└──────────────────┘    └──────────────────┘
```

### Copy-on-Write (COW) — Critical Optimization

**fork() does NOT immediately copy all memory.** Instead:

1. Both parent and child share the **same physical pages**
2. Pages are marked **read-only**
3. When either process **writes** to a page, a **page fault** occurs
4. The OS then copies **only that page** — this is the "copy on write"

```
After fork() with COW:

Parent's Page Table          Physical Memory         Child's Page Table
┌──────────┐                ┌──────────┐            ┌──────────┐
│ VP0 → PP0│ ──────────────→│  Page 0  │←───────────│ VP0 → PP0│
│ VP1 → PP1│ ──────────────→│  Page 1  │←───────────│ VP1 → PP1│
│ VP2 → PP2│ ──────────────→│  Page 2  │←───────────│ VP2 → PP2│
└──────────┘  (read-only)   └──────────┘ (read-only)└──────────┘

After child writes to Page 1:

Parent's Page Table          Physical Memory         Child's Page Table
┌──────────┐                ┌──────────┐            ┌──────────┐
│ VP0 → PP0│ ──────────────→│  Page 0  │←───────────│ VP0 → PP0│
│ VP1 → PP1│ ──────────────→│  Page 1  │            │ VP1 → PP3│──→┌──────┐
│ VP2 → PP2│ ──────────────→│  Page 2  │←───────────│ VP2 → PP2│   │Page 3│
└──────────┘                └──────────┘            └──────────┘   │(copy)│
                                                                   └──────┘
```

### 🔥 Google Interview Insight
> **Q: Why does Unix use the fork()+exec() model instead of a single "create process" call?**
> 
> A: Separation of concerns. Between `fork()` and `exec()`, the child can:
> - **Redirect I/O** (set up pipes, redirect stdin/stdout)
> - **Change privileges** (drop root permissions)
> - **Set environment variables**
> - **Close unnecessary file descriptors**
> 
> This makes process creation highly flexible. The `posix_spawn()` call exists as a combined alternative for performance, but loses this flexibility.

### exec()

`exec()` replaces the current process image with a new program:

```c
// Child process typically does:
pid_t pid = fork();
if (pid == 0) {
    // Set up I/O redirection here if needed
    execvp("ls", (char*[]){"ls", "-la", NULL});
    // exec NEVER returns on success
    // If we get here, exec failed
    perror("exec failed");
    exit(1);
}
```

### Zombie and Orphan Processes

| Type | What happens | Problem | Solution |
|------|-------------|---------|----------|
| **Zombie** | Child exits but parent hasn't called `wait()` | PCB stays in memory, PID is wasted | Parent must call `wait()` or use `SIGCHLD` handler |
| **Orphan** | Parent exits before child | Child needs a parent | Init (PID 1) adopts orphans and calls `wait()` |

---

## 2.5 Threads

A **thread** is a lightweight unit of execution within a process. Threads within the same process share:

```
What threads SHARE:                What each thread has its OWN:
┌─────────────────────┐           ┌─────────────────────┐
│ ✅ Code (text)       │           │ 🔒 Thread ID          │
│ ✅ Global data       │           │ 🔒 Program Counter     │
│ ✅ Heap              │           │ 🔒 Register set        │
│ ✅ Open files        │           │ 🔒 Stack               │
│ ✅ Signal handlers   │           │ 🔒 Stack pointer       │
│ ✅ Address space     │           │ 🔒 Scheduling priority │
│ ✅ PID               │           │ 🔒 errno (thread-local)│
└─────────────────────┘           └─────────────────────┘
```

### Process vs Thread — Performance Comparison

| Operation | Process | Thread |
|-----------|---------|--------|
| **Creation** | ~10-100 ms | ~0.1-1 ms |
| **Context Switch** | ~1-10 μs (full TLB flush) | ~0.1-1 μs (no TLB flush) |
| **Memory** | Separate address space | Shared address space |
| **Communication** | IPC (pipes, sockets, shared memory) | Direct memory access |
| **Crash isolation** | ✅ One crash doesn't affect others | ❌ One crash kills all threads |

### Why Threads are Faster to Context-Switch

Process switch: Must flush TLB (translation lookaside buffer), switch page tables, potentially lose all cached address translations.

Thread switch (same process): Same address space → **no TLB flush**, just swap register set and stack pointer.

---

## 2.6 Threading Models

### User-Level Threads (Many-to-One)

```
User Space          Kernel Space
┌──────────┐       ┌────────────┐
│ T1 T2 T3 │──────→│  1 Kernel  │
│ (library  │       │   Thread   │
│  manages) │       └────────────┘
└──────────┘
```
- Thread library does scheduling, kernel sees one thread
- **Pro**: Fast create/switch (no syscall)
- **Con**: One blocking syscall blocks ALL threads; can't use multiple CPUs

### Kernel-Level Threads (One-to-One)

```
User Space          Kernel Space
┌──────────┐       ┌────────────┐
│    T1    │──────→│    KT1     │
│    T2    │──────→│    KT2     │
│    T3    │──────→│    KT3     │
└──────────┘       └────────────┘
```
- Each user thread maps to a kernel thread
- **Pro**: True parallelism, one blocking doesn't affect others
- **Con**: More overhead per thread (kernel resources)
- **Used by**: Linux (NPTL), Windows

### Many-to-Many

```
User Space          Kernel Space
┌──────────┐       ┌────────────┐
│ T1 T2 T3 │──────→│    KT1     │
│ T4 T5    │──────→│    KT2     │
└──────────┘       └────────────┘
```
- M user threads mapped to N kernel threads (M ≥ N)
- Best of both worlds, but complex to implement

### 🔥 Google Interview Insight
> **Q: Google uses Go extensively. How does Go's goroutine model relate to threading models?**
> 
> A: Go uses an **M:N threading model** (many-to-many). Goroutines are user-level threads multiplexed onto a smaller number of OS threads by Go's runtime scheduler. This gives:
> - **Lightweight creation** (~2KB stack vs ~1MB for OS threads)
> - **True parallelism** (multiple OS threads across CPUs)
> - **Efficient blocking** (runtime can move goroutines between OS threads)
> 
> This is why Go can handle millions of concurrent goroutines.

---

## 2.7 Context Switching — What Really Happens

A **context switch** is when the CPU switches from one process/thread to another.

### Step-by-Step Context Switch

```
Process A running                Process B ready
     │
     ▼
1. Interrupt/syscall occurs (e.g., timer interrupt)
     │
     ▼
2. Hardware saves minimal state
   (PC, stack pointer → kernel stack)
     │
     ▼
3. Kernel interrupt handler runs
     │
     ▼
4. Scheduler decides: "Switch to Process B"
     │
     ▼
5. Save FULL context of A:
   - All CPU registers (rax, rbx, ... r15)
   - Floating-point registers
   - Stack pointer
   - Program counter
   → Store in A's PCB
     │
     ▼
6. Load FULL context of B:
   - Restore all registers from B's PCB
   - Switch page table (cr3 register on x86)
   - Flush TLB (if different process)
     │
     ▼
7. Return to user mode → Process B runs
```

### Context Switch Overhead

```
╔══════════════════════════════════════════════════╗
║  Context Switch Costs (~1-10 μs total)           ║
╠══════════════════════════════════════════════════╣
║  Direct costs:                                   ║
║   • Save/restore registers        ~100 ns        ║
║   • Switch page tables            ~50 ns         ║
║   • Kernel scheduling decision    ~100 ns        ║
║                                                  ║
║  Indirect costs (MUCH larger):                   ║
║   • TLB misses after flush        ~many μs       ║
║   • CPU cache misses (cold cache) ~many μs       ║
║   • Pipeline flush                ~10s of ns     ║
║   • Branch predictor pollution    ~variable      ║
╚══════════════════════════════════════════════════╝
```

The **indirect costs** (cache/TLB pollution) are often 10-100x more expensive than the direct costs.

---

## 2.8 Inter-Process Communication (IPC)

Since processes have separate address spaces, they need special mechanisms to communicate:

| Mechanism | Speed | Use Case | Complexity |
|-----------|-------|----------|------------|
| **Pipe** | Fast | Parent-child, linear data flow | Low |
| **Named Pipe (FIFO)** | Fast | Unrelated processes | Low |
| **Shared Memory** | Fastest | High-throughput data sharing | High (need sync) |
| **Message Queue** | Medium | Structured messages | Medium |
| **Socket** | Variable | Network/cross-machine | Medium |
| **Signal** | Fast | Notifications (not data) | Low |

### Pipes Example

```c
int pipefd[2];  // pipefd[0] = read end, pipefd[1] = write end
pipe(pipefd);

if (fork() == 0) {
    // Child: reads from pipe
    close(pipefd[1]);  // Close unused write end
    char buf[100];
    read(pipefd[0], buf, sizeof(buf));
    printf("Child received: %s\n", buf);
    close(pipefd[0]);
} else {
    // Parent: writes to pipe
    close(pipefd[0]);  // Close unused read end
    write(pipefd[1], "Hello from parent!", 18);
    close(pipefd[1]);
    wait(NULL);
}
```

### How Shell Pipes Work: `ls | grep foo | wc -l`

```
Shell creates:
  pipe1[0,1]  pipe2[0,1]

┌────┐ stdout→pipe1[1]  ┌──────┐ stdout→pipe2[1]  ┌──────┐
│ ls │──────────────────→│ grep │──────────────────→│  wc  │
└────┘   pipe1[0]→stdin  └──────┘   pipe2[0]→stdin  └──────┘

Each command runs in its own process.
The shell uses fork+exec+dup2 to wire it all up.
```

---

## 📝 Interview Questions — Test Yourself

### Conceptual
1. What is the difference between a process and a thread?
2. Explain Copy-on-Write in `fork()`. Why is it important?
3. What is a zombie process? How do you prevent them?
4. Why is context switching between threads faster than between processes?
5. Compare pipes vs shared memory for IPC. When would you choose each?

### Coding / Design
6. Write a program that creates a zombie process. Then fix it.
7. Implement a simple shell that supports pipes (e.g., `ls | grep foo`).
8. Design a thread pool. What data structures do you need?

### Deep Dive
9. Linux's `clone()` system call unifies process and thread creation. How?
10. What is `futex` and why is it important for modern threading?
11. Explain how `pthread_create()` works under the hood on Linux.

### Answers Guide

<details>
<summary>Click to reveal answers</summary>

**1.** A process is an independent execution unit with its own address space. A thread is a lightweight execution unit within a process, sharing the address space with other threads. Processes are isolated; threads share memory.

**2.** COW means `fork()` doesn't copy memory immediately. Parent and child share the same physical pages (marked read-only). Only when one writes does the OS copy that specific page. This is critical because most `fork()` calls are followed by `exec()`, which replaces all memory anyway — so copying would be wasted work.

**3.** A zombie is a child process that has exited but its parent hasn't called `wait()` to read its exit status. Prevention: (a) always `wait()` for children, (b) use `SIGCHLD` handler with `SA_NOCLDWAIT`, (c) double-fork trick (child forks again and exits, grandchild is adopted by init).

**4.** Thread switches within the same process don't need to: switch page tables (cr3), flush TLB, or invalidate process-specific caches. They only swap register sets and stack pointers.

**5.** Pipes: Simple, unidirectional, built-in flow control and synchronization. Best for producer-consumer with moderate data. Shared memory: Fastest (no kernel copying), bidirectional, but requires explicit synchronization (mutex/semaphore). Best for high-throughput, random-access data sharing.

**9.** `clone()` takes flags that control what is shared between parent and child:
- `CLONE_VM`: Share address space (thread-like)
- `CLONE_FILES`: Share file descriptor table
- `CLONE_FS`: Share filesystem info
- `CLONE_THREAD`: Share PID (appear as same process)
- `fork()` = `clone()` with no sharing flags
- `pthread_create()` = `clone()` with all sharing flags

**10.** `futex` (Fast Userspace muTEX) is a Linux syscall that enables efficient locking. In the uncontended case, the lock/unlock happens entirely in userspace (atomic compare-and-swap). Only when there's contention does it make a syscall to put the thread to sleep. This avoids the syscall overhead in the common case (no contention).

**11.** `pthread_create()` → calls `clone()` with flags `CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD` → kernel creates a new `task_struct` sharing the parent's address space → allocates a new kernel stack → new thread starts executing the specified function.

</details>

---

*Next: [Module 3: CPU Scheduling →](../03-CPU-Scheduling/README.md)*
