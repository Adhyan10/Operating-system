# Module 4: Process Synchronization

## 🧠 What You'll Learn
- The critical section problem and race conditions
- Mutex locks, semaphores, and monitors
- Classic synchronization problems (Producer-Consumer, Readers-Writers, Dining Philosophers)
- Spinlocks vs blocking locks
- Condition variables and their correct usage
- Lock-free programming basics

> **This is THE most important module for Google interviews.** Concurrency bugs are subtle, and Google builds massively concurrent systems. Expect deep questions here.

---

## 4.1 The Critical Section Problem

A **race condition** occurs when multiple threads access shared data and the outcome depends on the **order of execution**.

### Example: Bank Account Race Condition

```c
// Shared variable
int balance = 1000;

// Thread A: Withdraw 200
void withdraw(int amount) {
    int temp = balance;     // 1. Read balance (1000)
    temp = temp - amount;   // 2. Compute new balance (800)
    balance = temp;         // 3. Write back
}

// Thread B: Deposit 500
void deposit(int amount) {
    int temp = balance;     // 1. Read balance (1000) ← STILL 1000!
    temp = temp + amount;   // 2. Compute new balance (1500)
    balance = temp;         // 3. Write back (1500) ← OVERWRITES A's work!
}
```

```
Timeline showing the race:
Thread A                    Thread B                    balance
────────                    ────────                    ───────
read balance (1000)                                     1000
                            read balance (1000)         1000
temp = 1000 - 200 = 800                                1000
                            temp = 1000 + 500 = 1500   1000
write balance = 800                                     800
                            write balance = 1500        1500 ← WRONG!

Expected: 1000 - 200 + 500 = 1300
Got: 1500 (the $200 withdrawal was lost!)
```

### The Critical Section

```
Entry Section       ← Acquire permission to enter
┌──────────────┐
│   Critical   │    ← Only ONE thread at a time
│   Section    │       Access shared resources here
└──────────────┘
Exit Section        ← Release permission
Remainder Section   ← Non-critical code
```

### Three Requirements for a Correct Solution

| Requirement | Description |
|------------|-------------|
| **Mutual Exclusion** | At most one thread in the critical section at a time |
| **Progress** | If no thread is in CS, a waiting thread must eventually enter (no deadlock) |
| **Bounded Waiting** | A thread cannot wait forever (no starvation) |

---

## 4.2 Hardware Support for Synchronization

### Atomic Operations

Modern CPUs provide atomic (indivisible) instructions:

```c
// Test-and-Set (TAS) — atomic instruction
bool test_and_set(bool *target) {
    bool old = *target;
    *target = true;
    return old;
    // This ENTIRE operation is atomic — cannot be interrupted
}

// Using TAS for a spinlock:
bool lock = false;

void acquire() {
    while (test_and_set(&lock)) {
        // Spin (busy wait)
    }
    // Lock acquired!
}

void release() {
    lock = false;
}
```

### Compare-and-Swap (CAS) — The Foundation of Lock-Free Programming

```c
// CAS — atomic instruction
bool compare_and_swap(int *ptr, int expected, int new_value) {
    if (*ptr == expected) {
        *ptr = new_value;
        return true;   // Success
    }
    return false;      // Failed — someone else modified it
}

// Using CAS for a lock:
int lock = 0;

void acquire() {
    while (!compare_and_swap(&lock, 0, 1)) {
        // Spin
    }
}

void release() {
    lock = 0;
}
```

### 🔥 Google Interview Insight
> **Q: What's the ABA problem with CAS?**
> 
> A: Thread 1 reads value A, gets preempted. Thread 2 changes A→B→A. Thread 1 wakes up, sees A, CAS succeeds — but the state may have changed! Solution: Use a **version counter** alongside the value (e.g., CAS on {value, version} pair). This is critical for lock-free data structures.

---

## 4.3 Mutex (Mutual Exclusion Lock)

A **mutex** provides mutual exclusion: only one thread can hold it at a time.

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
int balance = 1000;

void withdraw(int amount) {
    pthread_mutex_lock(&lock);    // ACQUIRE — blocks if locked
    // ─── Critical Section ───
    balance -= amount;
    // ─── End Critical Section ───
    pthread_mutex_unlock(&lock);  // RELEASE
}

void deposit(int amount) {
    pthread_mutex_lock(&lock);
    balance += amount;
    pthread_mutex_unlock(&lock);
}
```

### Spinlock vs Mutex (Blocking Lock)

```
Spinlock:                         Mutex:
┌─────────────────────┐          ┌─────────────────────┐
│ while (locked) {    │          │ if (locked) {        │
│     // busy wait    │          │     sleep(thread);   │
│     // burns CPU!   │          │     // add to wait   │
│ }                   │          │     // queue         │
│                     │          │ }                    │
└─────────────────────┘          └─────────────────────┘
```

| Aspect | Spinlock | Blocking Mutex |
|--------|----------|---------------|
| **Wait mechanism** | Busy-wait (spin) | Sleep/wake |
| **CPU usage while waiting** | 100% | 0% |
| **Context switch** | No | Yes (expensive) |
| **Best for** | Short critical sections (<2μs) | Long critical sections |
| **Multi-CPU** | Works well | Works well |
| **Single CPU** | Terrible (wastes the only CPU) | Good |
| **Used in** | Kernel, low-level | User-space applications |

### 🔥 Google Interview Insight
> **Q: When would you use a spinlock over a mutex?**
> 
> A: Spinlocks when: (1) the critical section is very short (shorter than context switch overhead), (2) you're on a multiprocessor system (spinning on a single CPU wastes the only CPU), (3) you can't sleep (e.g., inside an interrupt handler in kernel code). In user-space, prefer mutexes. Linux's `futex` is a hybrid — spins briefly, then blocks.

---

## 4.4 Semaphores

A **semaphore** is a generalized lock with a counter. Invented by Dijkstra.

```
Semaphore S = integer value

wait(S) / P(S) / down(S):      signal(S) / V(S) / up(S):
    while (S <= 0)                  S++;
        block();                    wake_one_waiting();
    S--;
```

### Types of Semaphores

| Type | Initial Value | Purpose |
|------|--------------|---------|
| **Binary (0 or 1)** | 1 | Same as mutex |
| **Counting** | N | Allow up to N concurrent accessors |

### Semaphore for Resource Limiting

```c
// Database connection pool with max 5 connections
sem_t pool;
sem_init(&pool, 0, 5);  // Allow 5 concurrent connections

void use_connection() {
    sem_wait(&pool);     // Decrements. Blocks if count = 0
    // ─── Use connection ───
    do_database_work();
    // ─── Done ───
    sem_post(&pool);     // Increments. Wakes waiting thread
}
```

### Semaphore for Ordering / Signaling

```c
// Ensure Thread B runs AFTER Thread A completes its task
sem_t done;
sem_init(&done, 0, 0);  // Initially 0 — blocked

// Thread A:
void thread_a() {
    do_work();
    sem_post(&done);     // Signal: "I'm done!"
}

// Thread B:
void thread_b() {
    sem_wait(&done);     // Wait until A signals
    use_result();
}
```

---

## 4.5 Classic Synchronization Problems

### Problem 1: Producer-Consumer (Bounded Buffer)

```
┌─────────┐                        ┌─────────┐
│Producer │ ──→ [B U F F E R] ──→  │Consumer │
│         │     ↑ bounded ↑        │         │
└─────────┘     size = N           └─────────┘

Constraints:
- Producer must wait if buffer is FULL
- Consumer must wait if buffer is EMPTY
- Only one thread accesses buffer at a time
```

```c
#define BUFFER_SIZE 10
int buffer[BUFFER_SIZE];
int in = 0, out = 0;

sem_t empty;    // counts empty slots
sem_t full;     // counts full slots
pthread_mutex_t mutex;

void init() {
    sem_init(&empty, 0, BUFFER_SIZE);  // N empty slots
    sem_init(&full, 0, 0);             // 0 full slots
    pthread_mutex_init(&mutex, NULL);
}

void *producer(void *arg) {
    while (1) {
        int item = produce_item();
        
        sem_wait(&empty);              // Wait for empty slot
        pthread_mutex_lock(&mutex);    // Exclusive buffer access
        
        buffer[in] = item;
        in = (in + 1) % BUFFER_SIZE;
        
        pthread_mutex_unlock(&mutex);
        sem_post(&full);               // Signal: one more full slot
    }
}

void *consumer(void *arg) {
    while (1) {
        sem_wait(&full);               // Wait for full slot
        pthread_mutex_lock(&mutex);    // Exclusive buffer access
        
        int item = buffer[out];
        out = (out + 1) % BUFFER_SIZE;
        
        pthread_mutex_unlock(&mutex);
        sem_post(&empty);              // Signal: one more empty slot
        
        consume_item(item);
    }
}
```

> ⚠️ **Order matters!** If you `lock(mutex)` before `wait(empty)`, you can **deadlock**: producer holds mutex, waits for empty; consumer needs mutex to signal empty.

### Problem 2: Readers-Writers

```
Multiple readers can read simultaneously.
Writers need exclusive access.

Reader:                    Writer:
┌────────────────┐        ┌────────────────┐
│ Can share with │        │ Must be alone   │
│ other readers  │        │ No readers      │
│                │        │ No writers      │
└────────────────┘        └────────────────┘
```

**First Readers-Writers Solution (Readers Preference)**:

```c
int read_count = 0;           // Number of active readers
pthread_mutex_t mutex;        // Protects read_count
sem_t rw_lock;               // Exclusive access for writing

void reader() {
    pthread_mutex_lock(&mutex);
    read_count++;
    if (read_count == 1)      // First reader locks out writers
        sem_wait(&rw_lock);
    pthread_mutex_unlock(&mutex);
    
    // ─── Read shared data ───
    do_reading();
    
    pthread_mutex_lock(&mutex);
    read_count--;
    if (read_count == 0)      // Last reader lets writers in
        sem_post(&rw_lock);
    pthread_mutex_unlock(&mutex);
}

void writer() {
    sem_wait(&rw_lock);       // Exclusive access
    
    // ─── Write shared data ───
    do_writing();
    
    sem_post(&rw_lock);
}
```

**Problem**: Writers can starve if readers keep arriving!

**Solution — Writer-preference or fair (FIFO) approach**:
```c
// Add a "service" semaphore that ensures FIFO ordering
sem_t service;  // init to 1

void reader() {
    sem_wait(&service);       // Wait your turn in line
    
    pthread_mutex_lock(&mutex);
    read_count++;
    if (read_count == 1)
        sem_wait(&rw_lock);
    pthread_mutex_unlock(&mutex);
    
    sem_post(&service);       // Let next in line proceed
    
    do_reading();
    
    pthread_mutex_lock(&mutex);
    read_count--;
    if (read_count == 0)
        sem_post(&rw_lock);
    pthread_mutex_unlock(&mutex);
}

void writer() {
    sem_wait(&service);       // Wait your turn in line
    sem_wait(&rw_lock);       // Exclusive access
    sem_post(&service);       // Let next in line proceed
    
    do_writing();
    
    sem_post(&rw_lock);
}
```

### Problem 3: Dining Philosophers

```
5 philosophers sit around a table with 5 forks.
Each needs TWO forks (left and right) to eat.

        P0
      /    \
    F4      F0
    /        \
  P4    🍝    P1
    \        /
    F3      F1
      \    /
        P3
      /    \
    F2      
      \
        P2
```

**Naive (DEADLOCKING) solution**:
```c
// DANGER: This can deadlock!
void philosopher(int id) {
    while (1) {
        think();
        pick_up(fork[id]);           // Left fork
        pick_up(fork[(id+1) % 5]);   // Right fork
        // ↑ If ALL philosophers pick up left fork simultaneously,
        //   nobody can get right fork → DEADLOCK!
        eat();
        put_down(fork[id]);
        put_down(fork[(id+1) % 5]);
    }
}
```

**Solutions**:

| Solution | How it works |
|----------|-------------|
| **Asymmetric** | One philosopher picks up right fork first | Breaks circular wait |
| **Limit concurrency** | Allow at most 4 to sit at once (semaphore) |
| **Resource ordering** | Always pick up lower-numbered fork first |
| **Try-lock** | Use trylock; if second fork fails, release first, retry |

```c
// Solution: Resource ordering (always pick up lower-numbered fork first)
void philosopher(int id) {
    int first = min(id, (id+1) % 5);
    int second = max(id, (id+1) % 5);
    
    while (1) {
        think();
        pick_up(fork[first]);    // Always lower-numbered first
        pick_up(fork[second]);   // Then higher-numbered
        eat();
        put_down(fork[second]);
        put_down(fork[first]);
    }
}
```

---

## 4.6 Monitors and Condition Variables

A **monitor** is a higher-level synchronization construct that bundles:
- Shared data
- Operations on that data
- Automatic mutual exclusion
- Condition variables for signaling

### Condition Variables

```c
pthread_mutex_t mutex;
pthread_cond_t cond;
bool data_ready = false;

// Thread A: Waiter
void *waiter(void *arg) {
    pthread_mutex_lock(&mutex);
    while (!data_ready) {          // ALWAYS use while, not if!
        pthread_cond_wait(&cond, &mutex);
        // wait() atomically: releases mutex + sleeps
        // when woken: reacquires mutex
    }
    // Use data
    pthread_mutex_unlock(&mutex);
}

// Thread B: Signaler
void *signaler(void *arg) {
    pthread_mutex_lock(&mutex);
    data_ready = true;
    pthread_cond_signal(&cond);    // Wake one waiter
    pthread_mutex_unlock(&mutex);
}
```

### Why `while` and not `if`?

```
if (!data_ready) {            while (!data_ready) {
    cond_wait(&cond, &mutex);     cond_wait(&cond, &mutex);
}                              }

Problem with "if":            "while" handles:
- Spurious wakeups            ✅ Spurious wakeups
- Multiple waiters where      ✅ Stolen conditions
  another thread consumed
  the data first
```

### 🔥 Google Interview Insight
> **Q: Implement a thread-safe bounded queue using mutex + condition variables.**
> 
> This is a VERY common Google interview question:

```c
typedef struct {
    int buffer[MAX_SIZE];
    int head, tail, count;
    pthread_mutex_t lock;
    pthread_cond_t not_full;
    pthread_cond_t not_empty;
} BlockingQueue;

void enqueue(BlockingQueue *q, int item) {
    pthread_mutex_lock(&q->lock);
    while (q->count == MAX_SIZE) {
        pthread_cond_wait(&q->not_full, &q->lock);
    }
    q->buffer[q->tail] = item;
    q->tail = (q->tail + 1) % MAX_SIZE;
    q->count++;
    pthread_cond_signal(&q->not_empty);
    pthread_mutex_unlock(&q->lock);
}

int dequeue(BlockingQueue *q) {
    pthread_mutex_lock(&q->lock);
    while (q->count == 0) {
        pthread_cond_wait(&q->not_empty, &q->lock);
    }
    int item = q->buffer[q->head];
    q->head = (q->head + 1) % MAX_SIZE;
    q->count--;
    pthread_cond_signal(&q->not_full);
    pthread_mutex_unlock(&q->lock);
    return item;
}
```

---

## 4.7 Common Pitfalls

### 1. Lock Ordering Deadlock

```c
// Thread A:              Thread B:
lock(A);                  lock(B);    
lock(B);  ← waits         lock(A);  ← waits
// DEADLOCK!

// Fix: Always acquire locks in the same global order
// If you need A and B, ALWAYS lock(A) then lock(B)
```

### 2. Forgetting to Unlock

```c
void process() {
    pthread_mutex_lock(&lock);
    if (error_condition) {
        return;  // BUG! Lock is never released!
    }
    pthread_mutex_unlock(&lock);
}

// Fix: Use goto cleanup, or use RAII in C++:
void process() {
    std::lock_guard<std::mutex> guard(lock);  // Auto-unlock on scope exit
    if (error_condition) {
        return;  // guard destructor releases lock ✅
    }
}
```

### 3. Signal vs Broadcast

```
signal():    Wake ONE waiting thread
broadcast(): Wake ALL waiting threads

Use broadcast when:
- Multiple waiters might be able to proceed
- The condition change could satisfy multiple waiters
- You're not sure → broadcast is always safe (just less efficient)
```

---

## 📝 Interview Questions — Test Yourself

### Conceptual
1. What is a race condition? Give a real-world example.
2. Compare mutex vs semaphore. When would you use each?
3. Why must you use `while` instead of `if` with condition variables?
4. Explain the difference between `signal()` and `broadcast()`.

### Coding (Common Google Questions)
5. Implement a thread-safe singleton (double-checked locking).
6. Implement `print_in_order()` — three threads must print "first", "second", "third" in order.
7. Implement the H2O problem: threads represent H and O atoms and must form water molecules.
8. Implement a read-write lock from scratch using mutex + condition variables.
9. Implement a barrier that blocks N threads until all arrive.

### Deep Dive
10. What is priority inversion? How did it affect the Mars Pathfinder?
11. Explain the thundering herd problem and how to mitigate it.
12. What are memory barriers / fences? Why are they needed on modern CPUs?

### Answers Guide

<details>
<summary>Click to reveal answers</summary>

**5. Thread-safe Singleton (Double-Checked Locking)**:
```c
class Singleton {
    static std::atomic<Singleton*> instance;
    static std::mutex mtx;
    
    Singleton() {}
public:
    static Singleton* getInstance() {
        Singleton* tmp = instance.load(std::memory_order_acquire);
        if (tmp == nullptr) {
            std::lock_guard<std::mutex> lock(mtx);
            tmp = instance.load(std::memory_order_relaxed);
            if (tmp == nullptr) {
                tmp = new Singleton();
                instance.store(tmp, std::memory_order_release);
            }
        }
        return tmp;
    }
};
// In C++11+, just use: static Singleton& getInstance() {
//     static Singleton instance; return instance; } // Thread-safe by standard
```

**6. Print In Order**:
```c
sem_t s1, s2;
sem_init(&s1, 0, 0);
sem_init(&s2, 0, 0);

void first()  { print("first");  sem_post(&s1); }
void second() { sem_wait(&s1); print("second"); sem_post(&s2); }
void third()  { sem_wait(&s2); print("third"); }
```

**9. Barrier**:
```c
typedef struct {
    int n;           // total threads
    int count;       // arrived so far
    pthread_mutex_t lock;
    pthread_cond_t cv;
} Barrier;

void barrier_wait(Barrier *b) {
    pthread_mutex_lock(&b->lock);
    b->count++;
    if (b->count == b->n) {
        b->count = 0;  // reset for reuse
        pthread_cond_broadcast(&b->cv);  // wake ALL
    } else {
        while (b->count != 0) {  // haven't been reset yet
            pthread_cond_wait(&b->cv, &b->lock);
        }
    }
    pthread_mutex_unlock(&b->lock);
}
```

**10. Priority Inversion**: A high-priority task is blocked waiting for a lock held by a low-priority task. A medium-priority task preempts the low-priority task, indirectly blocking the high-priority task. Mars Pathfinder (1997): A high-priority bus task was blocked by a low-priority meteorological task holding a shared mutex. Medium-priority communication tasks kept preempting the met task, so the bus task missed its deadline and the watchdog timer reset the system repeatedly. Fix: **Priority inheritance** — temporarily boost the low-priority task to the high-priority task's level while it holds the lock.

**11. Thundering herd**: Multiple threads/processes are waiting on the same event. When the event occurs, ALL are woken up, but only one can proceed. The rest go back to sleep — wasting CPU on useless context switches. Mitigation: Use `signal()` instead of `broadcast()` when only one waiter can proceed. Linux's `accept()` was fixed to wake only one process.

**12. Memory barriers/fences force ordering of memory operations. Modern CPUs reorder loads and stores for performance. Without barriers, Thread A might write X=1 then flag=1, but another CPU might see flag=1 before X=1 (store reordering). Barriers ensure: all stores before the barrier are visible before stores after it. Critical for lock-free programming.

</details>

---

*Next: [Module 5: Deadlocks →](../05-Deadlocks/README.md)*
