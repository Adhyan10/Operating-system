# Module 11: Google Interview Problems — OS Edition

## 🎯 This Module

A curated set of problems that reflect the style, depth, and topics actually asked in Google interviews related to Operating Systems. Each problem includes the interview format it targets, hints, and detailed solutions.

---

## Category 1: Concurrency & Synchronization (Most Common!)

### Problem 1: Print FooBar Alternately

**Difficulty**: Medium | **Format**: Coding | **Frequency**: ⭐⭐⭐⭐⭐

Two threads: one prints "foo", the other prints "bar". Ensure output is "foobarfoobar..." for n iterations.

```c
// YOUR SOLUTION HERE — try before looking below!
```

<details>
<summary>Solution</summary>

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>

sem_t foo_sem, bar_sem;
int n = 5;

void* print_foo(void* arg) {
    for (int i = 0; i < n; i++) {
        sem_wait(&foo_sem);    // Wait for permission to print foo
        printf("foo");
        sem_post(&bar_sem);    // Signal bar to print
    }
    return NULL;
}

void* print_bar(void* arg) {
    for (int i = 0; i < n; i++) {
        sem_wait(&bar_sem);    // Wait for permission to print bar
        printf("bar");
        sem_post(&foo_sem);    // Signal foo to print
    }
    return NULL;
}

int main() {
    sem_init(&foo_sem, 0, 1);  // foo goes first
    sem_init(&bar_sem, 0, 0);  // bar waits
    
    pthread_t t1, t2;
    pthread_create(&t1, NULL, print_foo, NULL);
    pthread_create(&t2, NULL, print_bar, NULL);
    
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    
    printf("\n");
    return 0;
}
```

**Key insight**: Two semaphores create a "ping-pong" pattern. foo_sem starts at 1 (foo goes first), bar_sem starts at 0.
</details>

---

### Problem 2: Building H2O

**Difficulty**: Hard | **Format**: Coding | **Frequency**: ⭐⭐⭐⭐

Multiple threads, each representing an H or O atom. Group them into water molecules (2H + 1O). Atoms from the same molecule must bond together before any atom from the next molecule bonds.

```c
// Hint: Use a barrier to synchronize groups of 3
```

<details>
<summary>Solution</summary>

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>

sem_t h_sem;       // Controls hydrogen atoms
sem_t o_sem;       // Controls oxygen atoms
pthread_barrier_t barrier;  // Groups of 3
pthread_mutex_t mutex;

void hydrogen() {
    sem_wait(&h_sem);           // At most 2 H at a time
    pthread_barrier_wait(&barrier);  // Wait for group of 3
    printf("H");
    sem_post(&h_sem);           // Allow more H to enter (but controlled by O)
}

void oxygen() {
    sem_post(&h_sem);           // Allow 1 H
    sem_post(&h_sem);           // Allow 2 H
    sem_wait(&o_sem);           // Only 1 O at a time
    pthread_barrier_wait(&barrier);  // Wait for group of 3
    printf("O ");
    sem_post(&o_sem);
}
```

**Key insight**: The oxygen thread is the "organizer" — it posts two hydrogen permits and waits for the group.
</details>

---

### Problem 3: The Dining Philosophers (No Deadlock)

**Difficulty**: Medium | **Format**: Coding | **Frequency**: ⭐⭐⭐⭐

Implement the dining philosophers problem with guaranteed deadlock-freedom.

<details>
<summary>Solution — Resource Ordering</summary>

```c
#define N 5
pthread_mutex_t forks[N];

void philosopher(int id) {
    int left = id;
    int right = (id + 1) % N;
    
    // Always pick up lower-numbered fork first
    int first = (left < right) ? left : right;
    int second = (left < right) ? right : left;
    
    while (1) {
        think(id);
        
        pthread_mutex_lock(&forks[first]);
        pthread_mutex_lock(&forks[second]);
        
        eat(id);
        
        pthread_mutex_unlock(&forks[second]);
        pthread_mutex_unlock(&forks[first]);
    }
}
```

**Why it works**: Philosopher 4 picks up fork 0 before fork 4 (breaks circular wait). All others pick up left then right. No cycle possible.
</details>

---

### Problem 4: Implement a Read-Write Lock

**Difficulty**: Hard | **Format**: Coding + Design | **Frequency**: ⭐⭐⭐⭐

Implement a read-write lock with writer preference (no writer starvation).

<details>
<summary>Solution</summary>

```c
typedef struct {
    pthread_mutex_t mutex;
    pthread_cond_t readers_cv;
    pthread_cond_t writers_cv;
    int active_readers;
    int active_writers;
    int waiting_writers;
} RWLock;

void rwlock_init(RWLock *rw) {
    pthread_mutex_init(&rw->mutex, NULL);
    pthread_cond_init(&rw->readers_cv, NULL);
    pthread_cond_init(&rw->writers_cv, NULL);
    rw->active_readers = 0;
    rw->active_writers = 0;
    rw->waiting_writers = 0;
}

void read_lock(RWLock *rw) {
    pthread_mutex_lock(&rw->mutex);
    // Wait if there's an active writer OR waiting writers (writer preference!)
    while (rw->active_writers > 0 || rw->waiting_writers > 0) {
        pthread_cond_wait(&rw->readers_cv, &rw->mutex);
    }
    rw->active_readers++;
    pthread_mutex_unlock(&rw->mutex);
}

void read_unlock(RWLock *rw) {
    pthread_mutex_lock(&rw->mutex);
    rw->active_readers--;
    if (rw->active_readers == 0 && rw->waiting_writers > 0) {
        pthread_cond_signal(&rw->writers_cv);  // Wake one writer
    }
    pthread_mutex_unlock(&rw->mutex);
}

void write_lock(RWLock *rw) {
    pthread_mutex_lock(&rw->mutex);
    rw->waiting_writers++;
    while (rw->active_readers > 0 || rw->active_writers > 0) {
        pthread_cond_wait(&rw->writers_cv, &rw->mutex);
    }
    rw->waiting_writers--;
    rw->active_writers++;
    pthread_mutex_unlock(&rw->mutex);
}

void write_unlock(RWLock *rw) {
    pthread_mutex_lock(&rw->mutex);
    rw->active_writers--;
    if (rw->waiting_writers > 0) {
        pthread_cond_signal(&rw->writers_cv);  // Prefer writers
    } else {
        pthread_cond_broadcast(&rw->readers_cv);  // No writers? Wake all readers
    }
    pthread_mutex_unlock(&rw->mutex);
}
```

**Key decisions**:
- Writer preference: readers wait if writers are waiting
- `signal` for writers (only one can enter), `broadcast` for readers (all can enter)
</details>

---

### Problem 5: Implement a Thread-Safe Bounded Blocking Queue

**Difficulty**: Medium | **Format**: Coding | **Frequency**: ⭐⭐⭐⭐⭐

The single most common OS-related Google interview question!

<details>
<summary>Solution</summary>

```c
typedef struct {
    int *buffer;
    int capacity;
    int head, tail, count;
    pthread_mutex_t lock;
    pthread_cond_t not_full;
    pthread_cond_t not_empty;
} BoundedQueue;

BoundedQueue* queue_create(int capacity) {
    BoundedQueue *q = malloc(sizeof(BoundedQueue));
    q->buffer = malloc(sizeof(int) * capacity);
    q->capacity = capacity;
    q->head = q->tail = q->count = 0;
    pthread_mutex_init(&q->lock, NULL);
    pthread_cond_init(&q->not_full, NULL);
    pthread_cond_init(&q->not_empty, NULL);
    return q;
}

void queue_enqueue(BoundedQueue *q, int item) {
    pthread_mutex_lock(&q->lock);
    while (q->count == q->capacity) {       // FULL — wait
        pthread_cond_wait(&q->not_full, &q->lock);
    }
    q->buffer[q->tail] = item;
    q->tail = (q->tail + 1) % q->capacity;
    q->count++;
    pthread_cond_signal(&q->not_empty);     // Wake a consumer
    pthread_mutex_unlock(&q->lock);
}

int queue_dequeue(BoundedQueue *q) {
    pthread_mutex_lock(&q->lock);
    while (q->count == 0) {                 // EMPTY — wait
        pthread_cond_wait(&q->not_empty, &q->lock);
    }
    int item = q->buffer[q->head];
    q->head = (q->head + 1) % q->capacity;
    q->count--;
    pthread_cond_signal(&q->not_full);      // Wake a producer
    pthread_mutex_unlock(&q->lock);
    return item;
}
```

**Follow-up questions they'll ask**:
1. "Why `while` instead of `if`?" → Spurious wakeups + stolen conditions
2. "Why `signal` instead of `broadcast`?" → Only one thread can proceed anyway
3. "What if you need timeout?" → Use `pthread_cond_timedwait`
4. "Make it lock-free" → SPSC ring buffer for single-producer-single-consumer
</details>

---

## Category 2: Memory & System Design

### Problem 6: Design an LRU Cache

**Difficulty**: Medium | **Format**: Coding | **Frequency**: ⭐⭐⭐⭐⭐

Implement get(key) and put(key, value) in O(1).

<details>
<summary>Solution</summary>

```python
class Node:
    def __init__(self, key=0, val=0):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # key → Node
        self.head = Node()  # dummy head (MRU side)
        self.tail = Node()  # dummy tail (LRU side)
        self.head.next = self.tail
        self.tail.prev = self.head
    
    def _remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev
    
    def _add_to_front(self, node):
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node
    
    def get(self, key):
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)
        self._add_to_front(node)  # Move to MRU
        return node.val
    
    def put(self, key, value):
        if key in self.cache:
            self._remove(self.cache[key])
        node = Node(key, value)
        self.cache[key] = node
        self._add_to_front(node)
        
        if len(self.cache) > self.capacity:
            lru = self.tail.prev  # LRU node
            self._remove(lru)
            del self.cache[lru.key]
```

**Follow-up**: "Make it thread-safe" → Striped locking, or lock + RWLock for reads, or concurrent LRU with segmented lists.
</details>

---

### Problem 7: Design a Process Scheduler

**Difficulty**: Hard | **Format**: System Design | **Frequency**: ⭐⭐⭐

Design a process scheduler for a multi-core system.

<details>
<summary>Solution Approach</summary>

```
Requirements:
  - Support multiple priority levels
  - Handle both interactive and batch processes
  - Scale to 100+ CPUs
  - Minimize latency for interactive tasks

Design:
  
1. DATA STRUCTURES:
   - Per-CPU run queues (avoid global lock contention)
   - Each queue: red-black tree sorted by virtual runtime (like CFS)
   - Active and expired arrays for O(1) scheduling

2. SCHEDULING POLICY:
   - MLFQ for priority management
   - Virtual runtime with priority weighting (CFS-style)
   - Interactive boost: detect I/O-bound processes, keep them high priority
   
3. LOAD BALANCING:
   - Periodic (every 4ms): check load across CPUs
   - Pull migration: idle CPU steals from busiest
   - Push migration: overloaded CPU pushes to least loaded
   - NUMA-aware: prefer migrating within same NUMA node
   
4. REAL-TIME SUPPORT:
   - Highest priority: SCHED_FIFO/SCHED_RR for real-time
   - Below that: SCHED_NORMAL (CFS) for regular processes
   
5. FAIRNESS:
   - Track vruntime per process
   - Schedulable entity can be a process OR a cgroup
   - Two-level scheduling: fair between cgroups, fair within cgroup
```
</details>

---

### Problem 8: malloc Implementation

**Difficulty**: Hard | **Format**: Coding + Design | **Frequency**: ⭐⭐⭐

Design and implement a basic memory allocator.

<details>
<summary>Solution</summary>

```c
// Simplified malloc using free list + first-fit

typedef struct Block {
    size_t size;
    struct Block *next;
    int free;
} Block;

#define BLOCK_SIZE sizeof(Block)
#define ALIGN(size) (((size) + 7) & ~7)  // 8-byte alignment

static Block *free_list = NULL;

Block *find_free_block(size_t size) {
    Block *current = free_list;
    while (current) {
        if (current->free && current->size >= size) {
            return current;  // First fit
        }
        current = current->next;
    }
    return NULL;
}

Block *request_space(size_t size) {
    Block *block = sbrk(BLOCK_SIZE + size);
    if (block == (void*)-1) return NULL;
    block->size = size;
    block->free = 0;
    block->next = NULL;
    return block;
}

void *my_malloc(size_t size) {
    size = ALIGN(size);
    
    Block *block = find_free_block(size);
    if (block) {
        // Split if block is much larger
        if (block->size >= size + BLOCK_SIZE + 8) {
            Block *new_block = (Block*)((char*)block + BLOCK_SIZE + size);
            new_block->size = block->size - size - BLOCK_SIZE;
            new_block->free = 1;
            new_block->next = block->next;
            
            block->size = size;
            block->next = new_block;
        }
        block->free = 0;
        return (char*)block + BLOCK_SIZE;
    }
    
    block = request_space(size);
    if (!block) return NULL;
    
    // Add to free list
    if (!free_list) {
        free_list = block;
    } else {
        Block *last = free_list;
        while (last->next) last = last->next;
        last->next = block;
    }
    
    return (char*)block + BLOCK_SIZE;
}

void my_free(void *ptr) {
    if (!ptr) return;
    Block *block = (Block*)((char*)ptr - BLOCK_SIZE);
    block->free = 1;
    
    // Coalesce with next block if free
    if (block->next && block->next->free) {
        block->size += BLOCK_SIZE + block->next->size;
        block->next = block->next->next;
    }
}
```

**Google follow-ups**:
- "How would you make this thread-safe?" → Per-thread arenas (like TCMalloc)
- "How does TCMalloc work?" → Thread-local caches for small objects, central heap for large, size-class binning
- "What about fragmentation?" → Slab allocator, buddy system, or compaction
</details>

---

## Category 3: System Design — OS Concepts

### Problem 9: Design a File System

**Difficulty**: Hard | **Format**: System Design | **Frequency**: ⭐⭐⭐

<details>
<summary>Solution Approach</summary>

```
On-Disk Layout:
┌─────────┬──────────┬──────────┬──────────┬────────────┐
│Superblock│ Inode    │ Block    │ Inode    │   Data     │
│          │ Bitmap   │ Bitmap   │ Table    │   Blocks   │
└─────────┴──────────┴──────────┴──────────┴────────────┘

Key Design Decisions:

1. BLOCK SIZE: 4KB (matches page size, good for mmap)

2. INODE STRUCTURE:
   - 12 direct pointers (48KB without indirection)
   - 1 single indirect (4MB)
   - 1 double indirect (4GB)
   - Permissions, timestamps, size, link count

3. DIRECTORY:
   - Entries: {name, inode_number, type}
   - Use B-tree or hash for large directories

4. FREE SPACE:
   - Bitmap for both blocks and inodes
   - Group related data (block groups like ext4)

5. CRASH CONSISTENCY:
   - Write-ahead journaling
   - Journal metadata (ordered mode)
   - Checksum journal entries

6. CACHING:
   - Buffer cache for disk blocks
   - Inode cache
   - Dentry cache (name → inode mappings)
   - Page cache for file data

7. PERFORMANCE:
   - Extent-based allocation (contiguous blocks)
   - Delayed allocation (allocate on flush, not write)
   - Read-ahead (prefetch sequential blocks)
```
</details>

---

### Problem 10: Design a Container Runtime

**Difficulty**: Hard | **Format**: System Design | **Frequency**: ⭐⭐⭐

<details>
<summary>Solution Approach</summary>

```
Components:

1. FILESYSTEM ISOLATION:
   - OverlayFS: Stack read-only image layers + read-write container layer
   - pivot_root() to change root filesystem
   
2. PROCESS ISOLATION:
   - PID namespace: container has its own PID 1
   - clone(CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET | ...)
   
3. NETWORK ISOLATION:
   - Create veth pair (virtual ethernet)
   - One end in container namespace, one in host
   - Bridge on host for inter-container communication
   - iptables for NAT (container → internet)
   
4. RESOURCE LIMITS:
   - cgroups v2:
     - cpu.max: CPU time limit
     - memory.max: memory limit
     - io.max: disk I/O limit
     - pids.max: process count limit

5. SECURITY:
   - Drop unnecessary capabilities (cap_drop)
   - Seccomp-BPF: whitelist allowed syscalls
   - AppArmor/SELinux profiles
   - User namespace: map root in container to non-root on host

6. IMAGE FORMAT:
   - Layers: each layer is a tarball of filesystem changes
   - Manifest: lists layers, config, metadata
   - Content-addressable storage (SHA256)
   
7. LIFECYCLE:
   create → start → (running) → stop → remove
```
</details>

---

## Category 4: Quick-Fire Questions

These are often asked as warm-up or follow-up questions:

| # | Question | Key Points |
|---|----------|-----------|
| 1 | What happens when you type a URL in a browser? | DNS → TCP → HTTP → parse → render |
| 2 | What happens when you run `./program`? | Shell forks → child calls exec → kernel loads ELF → setup stack → jump to entry point |
| 3 | What is a context switch? | Save registers → switch page table → restore registers → TLB flush |
| 4 | Explain virtual memory in 2 minutes | Each process has private address space, OS+MMU translates to physical, pages can be on disk |
| 5 | What is a page fault? | Access to page not in memory → trap → OS loads from disk → restart instruction |
| 6 | How does `fork()` work? | Clone process, COW pages, returns 0 to child, child PID to parent |
| 7 | What is a deadlock? | Circular wait + mutual exclusion + hold-and-wait + no preemption |
| 8 | Mutex vs Semaphore | Mutex: ownership, binary. Semaphore: counting, no ownership, signaling |
| 9 | Process vs Thread | Process: separate address space. Thread: shared address space within process |
| 10 | What is thrashing? | Process spends more time paging than executing. Too many processes, not enough RAM |
| 11 | Stack vs Heap | Stack: auto, LIFO, fast, limited. Heap: manual, random access, slower, large |
| 12 | What is DMA? | Device transfers data directly to memory without CPU. One interrupt when done |
| 13 | User mode vs Kernel mode | User: restricted. Kernel: full access. Mode bit in CPU. Switch via syscall/interrupt |
| 14 | What is a system call? | User program requests kernel service. Mode switch + handler lookup + execute + return |
| 15 | Explain `mmap` | Map file/device into process address space. Access file as memory. Lazy loading via page faults |

---

## 📚 Recommended Study Order for Google Interview

```
Week 1: Modules 1-3 (Foundations, Processes, Scheduling)
Week 2: Module 4 (Synchronization) — PRACTICE CODING PROBLEMS
Week 3: Modules 5-6 (Deadlocks, Memory Management)
Week 4: Module 7-8 (File Systems, I/O)
Week 5: Modules 9-10 (Virtualization, Advanced Concurrency)
Week 6: Module 11 (this!) — Practice all interview problems

Key areas to focus (in order of interview frequency):
1. 🔥🔥🔥 Concurrency (mutex, semaphore, condition variables, coding)
2. 🔥🔥🔥 Memory Management (virtual memory, paging, LRU cache)
3. 🔥🔥  Process Management (fork, exec, threads)
4. 🔥🔥  Scheduling (CFS, MLFQ, trade-offs)
5. 🔥    File Systems (inodes, journaling)
6. 🔥    Virtualization/Containers (if applying for infra roles)
```

---

## 📖 Additional Resources

| Resource | Type | Best For |
|----------|------|----------|
| **OSTEP** (Operating Systems: Three Easy Pieces) | Free textbook | Fundamentals, clear explanations |
| **Linux Kernel Development** (Robert Love) | Book | Linux internals |
| **The Linux Programming Interface** (Kerrisk) | Book | System programming, syscalls |
| **xv6 Operating System** (MIT) | Teaching OS | Hands-on kernel code |
| **LeetCode Concurrency** | Problems | Coding practice |
| **Julia Evans' zines** | Visual | Quick reference |

---

*Good luck with your Google interview! 🚀*

*← [Back to Main Guide](../README.md)*
