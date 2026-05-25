# Module 10: Concurrency in Practice

## 🧠 What You'll Learn
- Memory ordering and memory models
- Lock-free and wait-free data structures
- Read-Copy-Update (RCU) — Linux's secret weapon
- Real-world concurrency bugs and how to avoid them
- Tools: ThreadSanitizer, Helgrind, Valgrind

> **This module bridges theory to practice.** Google interviews often ask about real-world concurrency patterns, not just textbook problems.

---

## 10.1 Memory Ordering — The Hidden Trap

Modern CPUs **reorder memory operations** for performance. This breaks naive concurrent code.

### The Problem

```c
// Initially: x = 0, y = 0

// Thread 1:              // Thread 2:
x = 1;                    y = 1;
r1 = y;                   r2 = x;

// Can r1 == 0 AND r2 == 0? On x86: NO. On ARM: YES!
```

### Why Reordering Happens

```
CPU Pipeline Optimization:

Without reordering:
  STORE x=1  [wait for cache line]  LOAD y [fast]
  Total: slow (store stalls pipeline)

With reordering:
  LOAD y [fast]  STORE x=1 [buffer it, write later]
  Total: fast! But other CPUs may see loads before stores.
```

### Memory Ordering Models

| Model | Strength | Example CPUs |
|-------|----------|-------------|
| **Sequential Consistency** | Strongest (no reordering) | Theoretical ideal |
| **Total Store Order (TSO)** | Only store-load reordering | x86, SPARC |
| **Relaxed** | Any reordering possible | ARM, RISC-V, PowerPC |

### Memory Barriers / Fences

Force ordering of memory operations:

```c
// Without barrier:
x = 1;          // might be reordered after load of y
r1 = y;

// With barrier:
x = 1;
__sync_synchronize();  // FULL MEMORY BARRIER
r1 = y;                // guaranteed to see x=1 first

// C11/C++11 atomics (preferred):
atomic_store_explicit(&x, 1, memory_order_release);
r1 = atomic_load_explicit(&y, memory_order_acquire);
```

### C++11 Memory Orders

```
memory_order_relaxed:   No ordering constraints. Only atomicity.
memory_order_acquire:   No reads/writes AFTER this can be reordered BEFORE it.
memory_order_release:   No reads/writes BEFORE this can be reordered AFTER it.
memory_order_acq_rel:   Both acquire and release.
memory_order_seq_cst:   Full sequential consistency (default, safest).

Common pattern — Release-Acquire:

Thread 1 (Producer):                Thread 2 (Consumer):
data = 42;                          while (!ready.load(acquire)) {}
ready.store(true, release);         assert(data == 42);  // GUARANTEED!
  ↑ release: all stores before        ↑ acquire: sees all stores
    this are visible                     before the release
```

### 🔥 Google Interview Insight
> **Q: What is the "happens-before" relationship?**
> 
> A: If operation A "happens-before" operation B, then A's effects are guaranteed visible to B. In C++:
> - Within a thread: sequential order = happens-before
> - Thread creation: parent's pre-create code happens-before child's execution
> - Mutex: `unlock()` happens-before subsequent `lock()`
> - Atomic: `release` store happens-before `acquire` load of the same variable
> 
> If two operations have no happens-before relationship AND at least one is a write, it's a **data race** = undefined behavior.

---

## 10.2 Lock-Free Data Structures

**Lock-free**: At least one thread makes progress in finite steps (no matter what other threads do).
**Wait-free**: ALL threads make progress in bounded steps.

### Lock-Free Stack (Treiber Stack)

```c
struct Node {
    int data;
    Node* next;
};

atomic<Node*> top = nullptr;

void push(int val) {
    Node* new_node = new Node{val, nullptr};
    Node* old_top;
    do {
        old_top = top.load(memory_order_relaxed);
        new_node->next = old_top;
    } while (!top.compare_exchange_weak(old_top, new_node,
                                         memory_order_release,
                                         memory_order_relaxed));
    // CAS: if top still == old_top, set top = new_node
    // If someone else changed top, retry with new old_top
}

int pop() {
    Node* old_top;
    Node* new_top;
    do {
        old_top = top.load(memory_order_acquire);
        if (old_top == nullptr) throw "empty";
        new_top = old_top->next;
    } while (!top.compare_exchange_weak(old_top, new_top,
                                         memory_order_release,
                                         memory_order_relaxed));
    int val = old_top->data;
    // Careful: can't delete old_top yet (ABA, other threads may read it)
    return val;
}
```

### ABA Problem (Revisited)

```
Thread 1: pop() reads top = A (A→B→C)
Thread 1: preempted

Thread 2: pop() A, pop() B, push() A (reuse A's memory)
           Now: top = A (but A→C, B is gone!)

Thread 1: resumes, CAS(top, A, B) succeeds!
           But B was already freed! → USE AFTER FREE

Solutions:
1. Hazard pointers: publish "I'm reading this, don't free it"
2. Epoch-based reclamation: defer deletion until safe
3. Double-width CAS with counter: CAS({ptr, version})
```

---

## 10.3 Read-Copy-Update (RCU)

Linux kernel's most important concurrency mechanism. Optimized for read-heavy, write-rare data.

### The Idea

```
Readers:                              Writer:
  Read shared data with NO locks!       1. Copy the data structure
  Just dereference a pointer.           2. Modify the copy
  Extremely fast.                       3. Atomically swap pointer
                                        4. Wait for all existing
                                           readers to finish
                                        5. Free old version

Timeline:
    Reader 1: ──read old──────done
    Reader 2: ────read old──────done
    Writer:   ──copy──modify──swap────wait──free old
    Reader 3:                   ──read new──done
```

### RCU Example

```c
// Shared data
struct Config {
    int timeout;
    int max_connections;
};
struct Config *global_config;

// READER (no lock!):
void reader() {
    rcu_read_lock();         // Marks start of critical section
                             // (just disables preemption, NO lock)
    struct Config *cfg = rcu_dereference(global_config);
    use(cfg->timeout);       // Safe to read
    rcu_read_unlock();       // Done reading
}

// WRITER (rare):
void update_config(int new_timeout) {
    struct Config *old = global_config;
    struct Config *new_cfg = kmalloc(sizeof(*new_cfg));
    *new_cfg = *old;                          // Copy
    new_cfg->timeout = new_timeout;           // Modify copy
    rcu_assign_pointer(global_config, new_cfg); // Swap (atomic)
    synchronize_rcu();                        // Wait for readers
    kfree(old);                               // Now safe to free
}
```

### Why RCU is Brilliant

| Aspect | Mutex | RCU |
|--------|-------|-----|
| **Read overhead** | Lock + unlock (~100ns) | ~0 (just preempt disable) |
| **Read scalability** | Poor (contention) | Perfect (no sharing) |
| **Write cost** | Low | Higher (copy + wait) |
| **Deadlock risk** | Yes | No (readers don't lock) |
| **Best for** | Write-heavy | Read-heavy (90%+ reads) |

Used extensively in: Linux routing tables, module lists, process lists, file system caches.

---

## 10.4 Common Concurrency Bugs

### Bug 1: Time-of-Check to Time-of-Use (TOCTOU)

```c
// VULNERABLE:
if (access("/tmp/file", R_OK) == 0) {   // CHECK
    // Attacker replaces /tmp/file with symlink here!
    fd = open("/tmp/file", O_RDONLY);     // USE
    // Now reading attacker's file!
}

// FIX: Use atomic operations
fd = open("/tmp/file", O_RDONLY);
if (fd >= 0) {
    // fstat(fd, ...) to verify properties
}
```

### Bug 2: Double-Checked Locking (Broken!)

```c
// BROKEN on most architectures:
Singleton* getInstance() {
    if (instance == NULL) {          // First check (no lock)
        lock(&mutex);
        if (instance == NULL) {      // Second check (with lock)
            instance = new Singleton();
            // Problem: compiler/CPU might reorder!
            // Other thread might see non-NULL instance
            // but with uninitialized fields!
        }
        unlock(&mutex);
    }
    return instance;
}

// FIX: Use atomic with acquire/release
Singleton* getInstance() {
    Singleton* tmp = instance.load(memory_order_acquire);
    if (tmp == NULL) {
        lock(&mutex);
        tmp = instance.load(memory_order_relaxed);
        if (tmp == NULL) {
            tmp = new Singleton();
            instance.store(tmp, memory_order_release);
        }
        unlock(&mutex);
    }
    return tmp;
}
```

### Bug 3: Lock-Order Inversion

```
Thread A: lock(database) → lock(logger)
Thread B: lock(logger) → lock(database)
→ DEADLOCK!

Detection: ThreadSanitizer builds a lock-order graph.
If it detects a cycle → reports potential deadlock.
```

---

## 10.5 Debugging Tools

| Tool | What it detects | How |
|------|----------------|-----|
| **ThreadSanitizer (TSan)** | Data races, lock-order violations | Compiler instrumentation |
| **AddressSanitizer (ASan)** | Buffer overflow, use-after-free | Shadow memory |
| **Helgrind** (Valgrind) | Data races, lock errors | Binary instrumentation |
| **DRD** (Valgrind) | Data races | Similar to Helgrind |
| **Intel Inspector** | Data races, deadlocks | Binary analysis |

```bash
# Compile with ThreadSanitizer:
gcc -fsanitize=thread -g -O1 program.c -o program
./program

# Output:
# WARNING: ThreadSanitizer: data race
#   Write of size 4 at 0x7f... by thread T1:
#     #0 increment() at program.c:15
#   Previous read of size 4 at 0x7f... by thread T2:
#     #0 read_counter() at program.c:22
```

---

## 10.6 Practical Patterns

### Pattern 1: Producer-Consumer with Lock-Free Queue

```
Use a single-producer single-consumer (SPSC) ring buffer:

┌───┬───┬───┬───┬───┬───┬───┬───┐
│ D │ D │ D │   │   │   │   │ D │
└───┴───┴───┴───┴───┴───┴───┴───┘
              ↑               ↑
            write           read
            (producer)      (consumer)

Producer only writes 'write' index.
Consumer only writes 'read' index.
No locks needed! Just atomic operations with appropriate ordering.
```

### Pattern 2: Read-Write Lock with Writer Preference

```
Use atomic counter:
  count > 0: count readers active
  count == 0: no readers
  count == -1: writer active

reader_lock:  while(CAS fails to increment positive count) spin;
reader_unlock: atomic_decrement
writer_lock:  while(CAS(0, -1) fails) spin;
writer_unlock: atomic_store(0)
```

### Pattern 3: Epoch-Based Memory Reclamation

```
For lock-free structures, can't free memory while readers exist.

Epochs: Global epoch counter (0, 1, 2)
  - Each thread records when it enters a read
  - Writer increments global epoch
  - Memory freed in epoch E is safe when ALL threads
    have progressed past epoch E

Thread enters read → record current epoch
Thread exits read → clear epoch
Reclaimer: check if all threads have advanced → safe to free
```

---

## 📝 Interview Questions — Test Yourself

### Conceptual
1. What is a memory barrier? Why is it needed on ARM but not on x86 for most patterns?
2. Explain the difference between lock-free and wait-free.
3. How does RCU work? When would you use it?
4. What is the ABA problem? How do you solve it?
5. What is a TOCTOU bug? Give an example.

### Coding
6. Implement a lock-free queue using CAS.
7. Implement a reader-writer lock using atomics (no mutex).
8. Write code that demonstrates a data race, then fix it.
9. Implement a simple epoch-based memory reclamation scheme.

### Design
10. Design a concurrent hash map. What are the trade-offs between striped locking, lock-free, and RCU approaches?
11. How would you design a thread-safe logger for a high-throughput system?

### Answers Guide

<details>
<summary>Click to reveal answers</summary>

**1.** A memory barrier forces ordering of memory operations. x86 has **Total Store Order (TSO)** — it only reorders stores after loads, which is the least common problematic pattern. ARM has a **relaxed** model — any operation can be reordered with any other. So on ARM, you need explicit barriers for most concurrent patterns. On x86, you mainly need barriers for the store-load case (e.g., `mfence`).

**10.** Concurrent hash map approaches:
- **Global lock**: Simplest, worst performance. One lock for entire map.
- **Striped locking**: Lock per bucket (or per N buckets). Good balance. Java's ConcurrentHashMap v7.
- **Lock-free**: Each bucket is a lock-free linked list. Complex, good for high contention. Use CAS for insert/delete.
- **RCU-based**: Readers never lock. Writers copy-on-write affected bucket. Best for read-heavy workloads. Linux kernel uses this extensively.

Trade-offs:
| Approach | Read Performance | Write Performance | Complexity |
|----------|-----------------|-------------------|-----------|
| Global lock | Poor | Poor | Simple |
| Striped | Good | Good | Medium |
| Lock-free | Excellent | Good | Hard |
| RCU | Best | Poor (copying) | Medium |

Google typically uses striped locking (Abseil's `flat_hash_map` with external locking) or custom lock-free structures for hot paths.

</details>

---

*Next: [Module 11: Google Interview Problems →](../11-Interview-Problems/README.md)*
