# Module 1: Foundations of Operating Systems

## 🧠 What You'll Learn
- What an OS actually does and why it exists
- Kernel vs User mode and privilege levels
- System calls — the gateway to the kernel
- How a computer boots up
- Interrupts and traps

---

## 1.1 What is an Operating System?

An Operating System is a **software layer** that sits between hardware and applications. It has two primary jobs:

1. **Resource Manager** — Manages CPU, memory, disk, and I/O devices fairly among programs
2. **Abstraction Provider** — Hides hardware complexity behind clean APIs (files, processes, sockets)

```
┌─────────────────────────────────────┐
│         User Applications           │
├─────────────────────────────────────┤
│          System Libraries           │  ← User Space
│         (libc, libpthread)          │
├─────────────────────────────────────┤
│     System Call Interface (API)     │  ← The Boundary
├─────────────────────────────────────┤
│          Operating System           │
│  ┌──────┬──────┬───────┬─────────┐  │
│  │ Proc │ Mem  │ File  │   I/O   │  │  ← Kernel Space
│  │ Mgmt │ Mgmt │ Sys   │  System │  │
│  └──────┴──────┴───────┴─────────┘  │
├─────────────────────────────────────┤
│            Hardware                 │
│   CPU  |  RAM  |  Disk  |  NIC     │
└─────────────────────────────────────┘
```

### Why does this matter for Google?
Google builds systems that run on millions of machines. Understanding what the OS does (and doesn't do) helps you reason about **performance**, **concurrency**, and **system design** — all critical interview topics.

---

## 1.2 Kernel Mode vs User Mode (Dual-Mode Operation)

The CPU has (at least) two privilege levels:

| Aspect | User Mode | Kernel Mode |
|--------|-----------|-------------|
| **Privilege** | Restricted | Full access |
| **Can access hardware?** | ❌ No | ✅ Yes |
| **Can execute privileged instructions?** | ❌ No (causes trap) | ✅ Yes |
| **Memory access** | Own address space only | All physical memory |
| **Who runs here?** | Applications | OS kernel |

### How the switch happens:

```
User Mode                          Kernel Mode
┌────────────┐                    ┌────────────────┐
│ Application│ ──system call──→   │  Kernel handler │
│   code     │ ←──return──────    │  (e.g. read())  │
└────────────┘                    └────────────────┘
                 ↑
          Mode bit flips
          (hardware support)
```

The CPU has a **mode bit** in a status register:
- `0` = Kernel mode
- `1` = User mode

When a system call or interrupt occurs, hardware **automatically** flips the mode bit to kernel mode and jumps to a predefined handler.

### 🔥 Google Interview Insight
> **Q: Why can't a user-mode program directly access disk?**
> 
> A: Because direct hardware access requires privileged instructions. If any program could write to any disk sector, one buggy app could corrupt another's data. The OS enforces **protection** by requiring all hardware access to go through **system calls**, where the kernel can validate permissions.

---

## 1.3 System Calls — The Gateway to the Kernel

A **system call** (syscall) is the mechanism by which a user program requests a service from the OS kernel.

### Common System Calls

| Category | System Calls | Purpose |
|----------|-------------|---------|
| **Process** | `fork()`, `exec()`, `wait()`, `exit()` | Create/manage processes |
| **File** | `open()`, `read()`, `write()`, `close()` | File I/O |
| **Memory** | `mmap()`, `brk()`, `sbrk()` | Memory allocation |
| **IPC** | `pipe()`, `shmget()`, `msgget()` | Inter-process communication |
| **Network** | `socket()`, `bind()`, `listen()`, `accept()` | Networking |

### How a System Call Works (Step by Step)

```
1. Application calls library function:  write(fd, buf, n)
                    │
2. Library (libc) puts syscall number   │
   in register (e.g., rax = 1 for      │
   write on x86-64)                     │
                    │
3. Execute SYSCALL instruction          │  ← Triggers trap
                    │
4. CPU switches to kernel mode          │  ← Mode bit = 0
   Saves user state (registers, PC)     │
                    │
5. Kernel looks up handler in           │
   syscall table[rax]                   │
                    │
6. Handler executes (e.g., writes       │
   to disk buffer)                      │
                    │
7. Kernel restores user state           │
   Sets return value in rax             │
                    │
8. SYSRET instruction                   │  ← Mode bit = 1
   Returns to user mode                 │
```

### Cost of a System Call

A system call is **expensive** compared to a normal function call:

| Operation | Approximate Cost |
|-----------|-----------------|
| Normal function call | ~1-5 ns |
| System call | ~100-1000 ns |
| Context switch (full) | ~1-10 μs |

**Why so expensive?**
- Mode switch (save/restore registers)
- TLB and cache pollution
- Pipeline flush on some architectures
- Kernel validation and permission checks

### 🔥 Google Interview Insight
> **Q: How would you optimize a program that makes too many system calls?**
> 
> A: Several strategies:
> 1. **Batching**: Use `writev()` instead of multiple `write()` calls
> 2. **Buffering**: Buffer data in user space (like `stdio` does with `printf`)
> 3. **Memory mapping**: Use `mmap()` to avoid read/write syscalls entirely
> 4. **io_uring** (Linux 5.1+): Submit batches of I/O requests asynchronously

---

## 1.4 Interrupts and Traps

The CPU doesn't just run your code — it constantly gets **interrupted**.

### Types of Interrupts

```
Interrupts
├── Hardware Interrupts (Asynchronous)
│   ├── Timer interrupt (clock tick)
│   ├── Keyboard/mouse input
│   ├── Disk I/O completion
│   └── Network packet arrival
│
└── Software Interrupts / Traps (Synchronous)
    ├── System calls (intentional trap)
    ├── Division by zero
    ├── Page fault
    └── Invalid opcode
```

### Interrupt Handling Flow

```
1. Device raises interrupt (sets IRQ line)
          │
2. CPU finishes current instruction
          │
3. CPU checks interrupt flag
          │
4. If enabled: save state → switch to kernel mode
          │
5. Look up handler in Interrupt Descriptor Table (IDT)
          │
6. Execute Interrupt Service Routine (ISR)
          │
7. ISR acknowledges interrupt to hardware
          │
8. Restore state → return to interrupted program
```

### Timer Interrupts — The OS's Heartbeat

The **timer interrupt** is arguably the most important interrupt. It fires at a regular interval (e.g., every 1-10ms) and is how the OS:

- **Preempts** running processes (enables multitasking)
- Updates system time
- Checks for sleeping processes to wake up
- Triggers scheduler decisions

> Without timer interrupts, a single while(true) loop would hang the entire system forever.

---

## 1.5 The Boot Process

Understanding boot helps you see how the OS takes control from hardware:

```
Power On
   │
   ▼
┌──────────────┐
│   BIOS/UEFI  │  Hardware initialization, POST (Power-On Self-Test)
│              │  Finds boot device
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Bootloader  │  GRUB/LILO loads the kernel into memory
│  (Stage 1+2) │  Sets up initial page tables
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Kernel     │  Initializes memory management, scheduler,
│  Init        │  device drivers, mounts root filesystem
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Init/systemd│  First user-space process (PID 1)
│              │  Starts system services
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Login/Shell │  User can now interact with the system
└──────────────┘
```

---

## 1.6 Monolithic vs Microkernel vs Hybrid

| Architecture | Description | Examples | Pros | Cons |
|-------------|-------------|----------|------|------|
| **Monolithic** | Everything in kernel space | Linux, FreeBSD | Fast (no IPC overhead) | Large, harder to maintain |
| **Microkernel** | Minimal kernel, services in user space | MINIX, QNX, seL4 | Reliable, modular | Slower (IPC overhead) |
| **Hybrid** | Mix of both approaches | Windows NT, macOS (XNU) | Balanced | Complex design |

```
Monolithic                    Microkernel
┌─────────────┐              ┌─────────────┐
│ User Space  │              │ User Space  │
├─────────────┤              │ ┌────┐┌────┐│
│ File System │              │ │FS  ││Net ││
│ Networking  │              │ │Srv ││Srv ││
│ Drivers     │              │ └────┘└────┘│
│ Scheduling  │              ├─────────────┤
│ Memory Mgmt │              │ IPC, Sched  │
├─────────────┤              │ Basic Mem   │
│  Hardware   │              ├─────────────┤
└─────────────┘              │  Hardware   │
                             └─────────────┘
```

### 🔥 Google Interview Insight
> **Q: Linux is monolithic but has loadable kernel modules. What's the advantage?**
> 
> A: Loadable modules give you **microkernel-like modularity** (add/remove drivers without reboot) while keeping **monolithic performance** (modules run in kernel space, no IPC overhead). It's the best of both worlds, but a buggy module can still crash the entire kernel.

---

## 📝 Interview Questions — Test Yourself

### Conceptual
1. What is the difference between a trap and an interrupt?
2. Why does the OS need dual-mode operation? What could go wrong without it?
3. Explain the lifecycle of a system call from user space to kernel and back.
4. Why is a system call more expensive than a regular function call?
5. What happens if the timer interrupt is disabled?

### Deep Dive
6. Compare `SYSCALL/SYSRET` (x86-64) with the older `INT 0x80` mechanism. Why is the newer one faster?
7. How does `vDSO` (virtual Dynamic Shared Object) in Linux avoid system calls for functions like `gettimeofday()`?
8. In a microkernel OS, how does a file read work compared to a monolithic kernel? Trace the IPC messages.

### Answers Guide

<details>
<summary>Click to reveal answers</summary>

**1.** A **trap** is synchronous — caused by the running instruction (syscall, division by zero). An **interrupt** is asynchronous — caused by external hardware (keyboard, timer, disk).

**2.** Without dual-mode, any program could execute privileged instructions (disable interrupts, access any memory, modify page tables). One buggy program could crash the entire system or read another program's data.

**3.** User calls library function → libc puts syscall number in register → executes trap instruction → CPU saves state, switches to kernel mode → kernel looks up handler in syscall table → handler executes → kernel puts result in register, restores state → returns to user mode.

**4.** A syscall requires: mode switch, register save/restore, potential TLB flush, pipeline flush, and kernel validation. A function call is just a stack push + jump.

**5.** The currently running process would monopolize the CPU forever. No other process would get to run. The OS scheduler would never execute. The system would appear "frozen" even though it's technically running one process.

**6.** `INT 0x80` goes through the full interrupt descriptor table lookup and is a general-purpose interrupt mechanism. `SYSCALL/SYSRET` is a purpose-built instruction pair that avoids the IDT lookup, doesn't change the stack to the kernel stack (the kernel does it manually), and is overall ~25-40% faster.

**7.** `vDSO` maps a small kernel-maintained page into every process's address space. For calls like `gettimeofday()`, the kernel updates this shared page on every timer tick. User programs read the time directly from this mapped page — no mode switch needed. It's essentially kernel data accessible in user mode (read-only).

**8.** In a microkernel: App → syscall to kernel → kernel sends IPC message to FS server (user space) → FS server processes request → sends IPC to disk driver (user space) → disk driver does I/O → sends result back via IPC → FS server sends result back → kernel delivers to app. That's at least 4 context switches vs 1 in a monolithic kernel.

</details>

---

*Next: [Module 2: Processes & Threads →](../02-Processes-and-Threads/README.md)*
