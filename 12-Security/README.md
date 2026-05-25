# Module 12: OS Security

## 🧠 What You'll Learn
- Memory attacks: buffer overflows, stack smashing, heap exploits
- OS defenses: ASLR, DEP/NX, stack canaries, KASLR
- Access control: DAC, MAC, capabilities
- Sandboxing and privilege separation
- Spectre/Meltdown — hardware vulnerabilities that broke the world

> **Google takes security extremely seriously.** Every engineer is expected to understand security fundamentals. In interviews, expect questions about why certain OS mechanisms exist and how they prevent attacks.

---

## 12.1 Memory Safety Attacks

### Buffer Overflow — The Classic Attack

A buffer overflow occurs when a program writes data **beyond the bounds** of a buffer, overwriting adjacent memory.

```c
// VULNERABLE code:
void vulnerable_function(char *input) {
    char buffer[64];
    strcpy(buffer, input);  // NO bounds checking!
    // If input > 64 bytes, it overwrites the stack!
}
```

### Stack Layout During Function Call

```
High Address
┌──────────────────────┐
│    Caller's Frame     │
├──────────────────────┤
│   Return Address      │ ← WHERE the function returns to
├──────────────────────┤
│   Saved Frame Pointer │ ← Old RBP (base pointer)
├──────────────────────┤
│                       │
│   buffer[64]          │ ← Our 64-byte buffer
│                       │
├──────────────────────┤
│   Local variables     │
└──────────────────────┘
Low Address

ATTACK: Overflow buffer to overwrite Return Address!
```

### Stack Smashing Attack Step-by-Step

```
Normal:                              After overflow:
┌──────────────────┐                ┌──────────────────┐
│ Return: 0x401000 │ (legit)       │ Return: 0xDEAD   │ ← OVERWRITTEN!
├──────────────────┤                ├──────────────────┤
│ Saved RBP        │                │ AAAAAAAAAAAAAAAA │ ← OVERWRITTEN!
├──────────────────┤                ├──────────────────┤
│ buffer[64]       │                │ AAAAAAAAAAAAAAAA │ ← Filled with
│ "Hello World"    │                │ AAAAAAAAAAAAAAAA │   attacker data
│                  │                │ AAAAAAAAAAAAAAAA │
├──────────────────┤                ├──────────────────┤
│ local vars       │                │ local vars       │
└──────────────────┘                └──────────────────┘

When function returns → jumps to 0xDEAD → attacker's code!

Classic payload: overwrite return address → jump to shellcode
(injected machine code that spawns a shell)
```

### Return-to-libc Attack

Even without injecting code, attacker can redirect execution to existing functions:

```
Instead of injecting shellcode:
  Overwrite return address → address of system() in libc
  Overwrite argument → address of "/bin/sh" string

Function returns → calls system("/bin/sh") → attacker gets shell!

This bypasses "no-execute" protections (DEP/NX).
```

### Heap Overflow

```
Heap allocator metadata corruption:

┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Chunk A      │  │ Chunk B      │  │ Chunk C      │
│ [header]     │  │ [header]     │  │ [header]     │
│ [user data]  │  │ [user data]  │  │ [user data]  │
│ OVERFLOW ────┼──┼→[corrupted!] │  │              │
└─────────────┘  └─────────────┘  └─────────────┘

Corrupting heap metadata can lead to:
  - Arbitrary write (write-what-where primitive)
  - Code execution when corrupted chunk is freed
  - Use-after-free exploitation
```

### Format String Attack

```c
// VULNERABLE:
printf(user_input);        // User controls format string!

// If user_input = "%x %x %x %x":
//   Prints values FROM THE STACK (information leak)

// If user_input = "%n":
//   WRITES to memory! (number of bytes printed so far)

// SAFE:
printf("%s", user_input);  // User input is just data, not format
```

### Use-After-Free

```c
char *ptr = malloc(100);
// ... use ptr ...
free(ptr);
// ... ptr is now dangling ...

// Attacker: allocate same-sized buffer, gets ptr's old memory
char *evil = malloc(100);
// evil might point to where ptr was!
// Anything ptr still references is now attacker-controlled

// Use of ptr now reads attacker's data:
process(ptr);  // BUG: use-after-free
```

---

## 12.2 OS Defenses

### Stack Canaries (Stack Protector)

```
Insert a random value ("canary") between buffer and return address:

┌──────────────────┐
│ Return Address    │
├──────────────────┤
│ Saved RBP        │
├──────────────────┤
│ ★ CANARY ★       │ ← Random value, checked before return
├──────────────────┤
│ buffer[64]       │
├──────────────────┤

Before function returns:
  if (canary != expected_value)
      abort();  // Buffer overflow detected!

Attacker can't overflow past the canary without corrupting it.
They don't know the canary value (randomized per process).

Compile with: gcc -fstack-protector-all program.c
```

### Address Space Layout Randomization (ASLR)

```
WITHOUT ASLR (predictable):          WITH ASLR (randomized):
┌──────────┐ 0xBFFF0000             ┌──────────┐ 0x7FFC????
│  Stack   │                        │  Stack   │ ← Random!
├──────────┤                        ├──────────┤
│          │                        │          │
├──────────┤ 0x40000000             ├──────────┤ 0x7F????
│  Heap    │                        │  Heap    │ ← Random!
├──────────┤                        ├──────────┤
│  libc    │ 0xB7E00000             │  libc    │ 0x7F????
├──────────┤                        ├──────────┤
│  Code    │ 0x08048000             │  Code    │ 0x55????
└──────────┘                        └──────────┘

Attacker can't predict where:
- Their shellcode lives (stack address unknown)
- system() is (libc address unknown)
- Gadgets are (code address unknown with PIE)

Check ASLR status: cat /proc/sys/kernel/randomize_va_space
  0 = off, 1 = partial, 2 = full
```

### Data Execution Prevention (DEP / NX Bit)

```
Mark memory pages as either EXECUTABLE or WRITABLE, never both:

Stack:  rw- (read-write, NOT executable)    ← Can't run shellcode here!
Heap:   rw- (read-write, NOT executable)    ← Can't run shellcode here!
Code:   r-x (read-execute, NOT writable)    ← Can't inject code here!

Hardware support: NX bit (No-eXecute) in page table entry.
CPU refuses to execute instructions from pages marked NX.

This defeats basic buffer overflow → shellcode attacks.
Attackers responded with ROP (Return-Oriented Programming).
```

### Return-Oriented Programming (ROP) — Defeating DEP

```
Instead of injecting code, REUSE existing code fragments ("gadgets"):

Gadgets: short instruction sequences ending in "ret"

Example gadgets in libc:
  Gadget 1: pop rdi; ret     (load argument)
  Gadget 2: pop rsi; ret     (load second argument)
  Gadget 3: address of system()

Attack: Chain gadgets on the stack:

Stack (overflow):
┌──────────────────┐
│ Gadget 1 addr    │ → pop rdi; ret
├──────────────────┤
│ "/bin/sh" addr   │ → loaded into rdi (first argument)
├──────────────────┤
│ system() addr    │ → called with rdi="/bin/sh"
└──────────────────┘

Result: Executes system("/bin/sh") using ONLY existing code!
No injected shellcode needed. DEP doesn't help.
Defense: ASLR (can't find gadgets if addresses randomized)
```

### 🔥 Google Interview Insight
> **Q: A process has ASLR, stack canaries, and DEP enabled. Is it safe from buffer overflows?**
> 
> A: **No, but it's much harder to exploit.** Attacks still possible via:
> 1. **Information leaks**: printf format strings or other bugs that leak addresses → defeats ASLR
> 2. **Brute force**: On 32-bit systems, ASLR has only ~16 bits of entropy → can be brute-forced
> 3. **Partial overwrites**: Overwrite only the low bytes of a pointer (low bytes aren't randomized)
> 4. **Heap attacks**: Canaries only protect the stack
> 5. **Side channels**: Spectre-style attacks can leak memory contents
> 
> Defense-in-depth is key. No single defense is sufficient.

---

## 12.3 Access Control

### Discretionary Access Control (DAC) — Unix Model

```
Every file has:
  Owner (UID), Group (GID), Permissions

Permission bits:
  rwx rwx rwx
  │   │   └── Others (everyone else)
  │   └────── Group
  └────────── Owner

Example: -rw-r--r-- 1 alice developers 4096 Jan 1 file.txt
  Owner (alice): read + write
  Group (developers): read only
  Others: read only

Special bits:
  SUID (4xxx): Run as file OWNER, not caller
    Example: /usr/bin/passwd is SUID root
    Regular user runs passwd → runs as root → can modify /etc/shadow
    
  SGID (2xxx): Run as file GROUP
  Sticky bit (1xxx): Only owner can delete files in directory
    Example: /tmp has sticky bit → users can't delete each other's files
```

### Mandatory Access Control (MAC) — SELinux/AppArmor

```
DAC problem: If root is compromised, game over.
MAC: Even root is constrained by policy.

SELinux labels:
  Every process and file has a security context:
  user:role:type:level
  
  Example:
    Process: system_u:system_r:httpd_t:s0  (Apache web server)
    File:    system_u:object_r:httpd_sys_content_t:s0  (/var/www/html)
    
  Policy rule:
    allow httpd_t httpd_sys_content_t:file { read };
    
  Result: Apache can ONLY read files labeled httpd_sys_content_t
  Even if Apache is running as root!
  Even if an attacker gets code execution in Apache!
```

### Linux Capabilities

```
Traditional: Root has ALL privileges. Any root exploit = total compromise.
Capabilities: Split root's powers into ~40 individual capabilities.

CAP_NET_BIND_SERVICE  → Bind to ports < 1024
CAP_SYS_ADMIN         → Mount filesystems, many admin ops
CAP_NET_RAW           → Use raw sockets (ping)
CAP_SYS_PTRACE        → Trace/debug other processes
CAP_CHOWN             → Change file ownership
CAP_DAC_OVERRIDE      → Bypass file permission checks
CAP_KILL              → Send signals to any process
CAP_SETUID            → Change process UID

Example: Give a web server ONLY port-binding capability:
  setcap 'cap_net_bind_service=+ep' /usr/bin/my_server
  
  Now my_server can bind port 80 without being root.
  If compromised, attacker can't mount filesystems, kill processes, etc.

Docker drops most capabilities by default:
  Only keeps ~14 of ~40 capabilities.
```

---

## 12.4 Sandboxing

### seccomp-BPF (System Call Filtering)

```
Restrict which system calls a process can make:

seccomp modes:
  STRICT: Only read(), write(), exit(), sigreturn()
  FILTER: Custom BPF program decides per-syscall

Example: Web browser renderer process
  Allowed: read, write, mmap, brk, futex
  Blocked: open, fork, exec, socket, mount, ...
  
  Even if attacker gets code execution in renderer:
    - Can't open files
    - Can't spawn processes
    - Can't make network connections
    - Can't access devices

Chrome uses seccomp-BPF for renderer sandboxing.
```

### Privilege Separation

```
Classic example: OpenSSH

┌─────────────────┐    ┌─────────────────┐
│  Privileged     │    │  Unprivileged   │
│  Monitor        │←──→│  Worker         │
│  (root, minimal │    │  (nobody, does  │
│   code)         │    │   all parsing)  │
└─────────────────┘    └─────────────────┘

Network input is parsed by the unprivileged worker.
If an exploit happens in parsing → attacker is "nobody" user.
Privileged operations go through a narrow, audited channel.
```

### chroot / pivot_root

```
chroot("/var/sandbox/");
// Now "/" refers to /var/sandbox/
// Process can't access files outside this directory
// (In theory — root can escape chroot. Containers use pivot_root instead.)
```

---

## 12.5 Spectre and Meltdown (Hardware Side-Channel Attacks)

These 2018 vulnerabilities shook the entire industry.

### Meltdown (CVE-2017-5754)

```
Exploits out-of-order execution to read KERNEL memory from user space.

Normal execution:
  User tries to read kernel address → permission check → DENIED (fault)
  
  But the CPU is pipelined and speculative:
  
  CPU Pipeline:
  1. Fetch: load kernel_data = *(kernel_addr)     ← Speculatively executes!
  2. Check: is this allowed? → NO
  3. Rollback: undo the load, raise exception
  
  BUT: The speculatively loaded data affected the CACHE!
  
  Attack:
  1. Access kernel address (will fault, but data is speculatively loaded)
  2. Use loaded byte as index into a probe array:
     probe_array[kernel_data * PAGE_SIZE]
  3. After fault is caught, measure which probe_array page is cached
  4. Fast access → that's the kernel data byte value!
  
  Result: Read arbitrary kernel memory at ~500KB/sec

Fix: KPTI (Kernel Page Table Isolation)
  Unmap kernel pages from user-space page table.
  Only a minimal trampoline remains mapped.
  Cost: ~5-30% performance hit (TLB flush on every syscall)
```

### Spectre (CVE-2017-5753, CVE-2017-5715)

```
Exploits branch prediction to leak data across security boundaries.

Variant 1 (Bounds Check Bypass):
  if (x < array1_size) {          // Branch predictor: "usually true"
      y = array2[array1[x] * 256]; // Speculatively executed with
  }                                //   out-of-bounds x!

  Train branch predictor with valid x values.
  Then provide malicious x (out of bounds).
  Branch predictor predicts "true" → speculatively reads array1[x]
  → uses result to index array2 → cache side channel → leaked data!

Affects: Cross-process, cross-VM, JavaScript in browsers!

Fixes:
  - Retpoline: replace indirect branches with a "return trampoline"
  - Microcode updates
  - Compiler barriers (lfence after bounds checks)
  - Site isolation in browsers (Chrome)
```

### 🔥 Google Interview Insight
> **Q: How did Spectre affect Google's infrastructure?**
> 
> A: Massively. Google runs multi-tenant cloud infrastructure where different customers' VMs share the same CPU.
> - **Chrome**: Implemented Site Isolation (each site in separate process) to prevent JavaScript Spectre attacks
> - **Cloud**: Had to apply microcode patches + retpoline to all servers
> - **Performance**: KPTI caused ~5% performance regression across the fleet
> - **Borg/Kubernetes**: Had to consider CPU co-tenancy in scheduling decisions
> - Google was one of Project Zero's discoverers of these vulnerabilities

---

## 12.6 Kernel Security Features Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    Defense-in-Depth                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Code Level:                                                │
│    ├── Stack canaries (detect stack overflow)                │
│    ├── FORTIFY_SOURCE (bounds-checked libc functions)        │
│    ├── -fstack-protector-strong                             │
│    └── Control Flow Integrity (CFI)                         │
│                                                             │
│  Memory Layout:                                             │
│    ├── ASLR (randomize addresses)                           │
│    ├── PIE (Position Independent Executables)               │
│    ├── DEP/NX (non-executable stack/heap)                   │
│    └── KASLR (kernel ASLR)                                  │
│                                                             │
│  Process Level:                                             │
│    ├── seccomp-BPF (syscall filtering)                      │
│    ├── Namespaces (visibility isolation)                    │
│    ├── Cgroups (resource limits)                            │
│    └── Capabilities (fine-grained privileges)               │
│                                                             │
│  OS Level:                                                  │
│    ├── SELinux / AppArmor (mandatory access control)        │
│    ├── KPTI (kernel page table isolation)                   │
│    ├── SMEP/SMAP (prevent kernel from executing user pages) │
│    └── Audit logging                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📝 Interview Questions — Test Yourself

### Conceptual
1. Explain a buffer overflow attack step by step.
2. How does ASLR work? What are its limitations?
3. What is DEP/NX? How does ROP defeat it?
4. Compare DAC and MAC. When is MAC necessary?
5. What are Linux capabilities? Why are they better than SUID?
6. Explain the Meltdown attack. How does KPTI fix it?

### Coding
7. Write a function that is vulnerable to buffer overflow. Then fix it.
8. Given a binary with ASLR but no canaries — describe an attack plan.

### Design
9. Design a sandboxing system for running untrusted user code (like LeetCode's judge).
10. How would you secure a container running untrusted workloads?

### Answers Guide

<details>
<summary>Click to reveal answers</summary>

**7.** Vulnerable:
```c
void vulnerable(char *input) {
    char buf[64];
    strcpy(buf, input);  // No bounds check
}
```
Fixed:
```c
void safe(char *input) {
    char buf[64];
    strncpy(buf, input, sizeof(buf) - 1);  // Bounds check
    buf[sizeof(buf) - 1] = '\0';           // Ensure null termination
}
// Or better: use strlcpy() or snprintf()
```

**9.** Sandboxing untrusted code (LeetCode judge):
- **Process isolation**: Fork a child process for each submission
- **User namespace**: Run as unprivileged user (not root)
- **seccomp-BPF**: Whitelist only needed syscalls (read, write, brk, mmap, exit)
- **Resource limits**: cgroup limits on CPU time (kill after 10s), memory (256MB max), processes (no fork bombs)
- **Network namespace**: No network access (empty namespace)
- **Mount namespace**: Read-only filesystem, only /lib and submission file
- **PID namespace**: Can't see or signal other processes
- **Time limit**: Alarm signal after timeout
- **Output limit**: Cap stdout/stderr to prevent disk fill
- **Additional**: Run in a minimal VM (Firecracker) for strongest isolation

**10.** Securing untrusted container workloads:
1. Drop all capabilities except needed ones
2. Run as non-root user (USER directive in Dockerfile)
3. Read-only root filesystem
4. seccomp profile blocking dangerous syscalls
5. AppArmor/SELinux profile
6. Resource limits via cgroups
7. No host network/PID namespace sharing
8. Consider gVisor (user-space kernel) or Kata containers (lightweight VM)
9. Network policies (restrict egress)
10. Image scanning for vulnerabilities

</details>

---

*Next: [Module 13: Signals & Networking →](../13-Signals-and-Networking/README.md)*
