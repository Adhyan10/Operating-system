# Module 13: Signals, Sockets & Networking at the OS Level

## 🧠 What You'll Learn
- Unix signals: how they work, how to handle them, common pitfalls
- Socket programming: how the OS implements networking
- The kernel networking stack: from NIC to application
- `select`, `poll`, `epoll`, `io_uring` — deep dive
- TCP state machine at the OS level

> **Signals are asked surprisingly often in Google interviews** (especially for SRE and systems roles). Networking at the OS level connects to system design interviews.

---

## 13.1 Unix Signals — OS Notifications to Processes

A **signal** is an asynchronous notification sent to a process by the OS or another process.

### Signal Lifecycle

```
Signal Generated                     Signal Delivered
     │                                     │
     │  ┌─────────────────────┐            │
     └──│ Signal is "pending" │────────────┘
        │ until delivered     │
        └─────────────────────┘
                    │
                    │ (process is scheduled, 
                    │  signal not blocked)
                    ▼
        ┌─────────────────────┐
        │ Signal Handler Runs │
        │ (or default action) │
        └─────────────────────┘
```

### Common Signals

| Signal | Number | Default Action | Purpose | Can Catch? |
|--------|--------|---------------|---------|-----------|
| `SIGHUP` | 1 | Terminate | Terminal hangup / config reload | ✅ |
| `SIGINT` | 2 | Terminate | Ctrl+C | ✅ |
| `SIGQUIT` | 3 | Core dump | Ctrl+\\ | ✅ |
| `SIGILL` | 4 | Core dump | Illegal instruction | ✅ |
| `SIGABRT` | 6 | Core dump | abort() called | ✅ |
| `SIGFPE` | 8 | Core dump | Floating point exception (div by 0) | ✅ |
| `SIGKILL` | 9 | Terminate | **Force kill (CANNOT be caught!)** | ❌ |
| `SIGSEGV` | 11 | Core dump | Segmentation fault (invalid memory) | ✅ |
| `SIGPIPE` | 13 | Terminate | Write to broken pipe | ✅ |
| `SIGALRM` | 14 | Terminate | Alarm timer expired | ✅ |
| `SIGTERM` | 15 | Terminate | Polite termination request | ✅ |
| `SIGCHLD` | 17 | Ignore | Child process stopped/terminated | ✅ |
| `SIGCONT` | 18 | Continue | Resume stopped process | ✅ |
| `SIGSTOP` | 19 | Stop | **Force stop (CANNOT be caught!)** | ❌ |
| `SIGTSTP` | 20 | Stop | Ctrl+Z | ✅ |
| `SIGUSR1` | 10 | Terminate | User-defined signal 1 | ✅ |
| `SIGUSR2` | 12 | Terminate | User-defined signal 2 | ✅ |

### 🔥 Google Interview Insight
> **Q: What's the difference between SIGKILL and SIGTERM? When do you use each?**
> 
> A: `SIGTERM` (15) is a **polite request** — the process can catch it, clean up (close files, release locks, notify children), and exit gracefully. `SIGKILL` (9) **cannot be caught or ignored** — the kernel immediately terminates the process. No cleanup happens.
> 
> **Always send SIGTERM first**, wait a few seconds, then SIGKILL if the process hasn't exited. This is exactly what `docker stop` does (SIGTERM → 10s grace → SIGKILL) and what Kubernetes uses for pod termination.
> 
> A process that can't be killed by SIGKILL is either: a zombie (already dead, parent hasn't waited), or stuck in uninterruptible sleep (D state, usually waiting for I/O from a dead NFS server).

---

## 13.2 Signal Handling

### Setting Up a Signal Handler

```c
#include <signal.h>
#include <stdio.h>
#include <unistd.h>

// Signal handler function
void handle_sigint(int sig) {
    // WARNING: Very limited operations safe here!
    write(STDOUT_FILENO, "Caught SIGINT!\n", 15);
    // printf is NOT safe in signal handlers!
}

int main() {
    // Modern way: sigaction (preferred over signal())
    struct sigaction sa;
    sa.sa_handler = handle_sigint;
    sigemptyset(&sa.sa_mask);    // Don't block other signals during handler
    sa.sa_flags = SA_RESTART;    // Restart interrupted syscalls
    
    sigaction(SIGINT, &sa, NULL);
    
    while (1) {
        printf("Running... (Ctrl+C to test)\n");
        sleep(1);
    }
    return 0;
}
```

### Async-Signal-Safe Functions

**Most library functions are NOT safe to call from signal handlers!**

```
SAFE in signal handlers:          UNSAFE in signal handlers:
✅ write()                        ❌ printf() (uses locks)
✅ _exit()                        ❌ malloc() (uses locks)
✅ signal()                       ❌ free() (uses locks)
✅ read()                         ❌ syslog()
✅ open(), close()                ❌ Any function using global state
✅ fork() (careful!)              ❌ strtok, localtime, etc.
✅ execve()

Why? Signal can arrive while printf() holds a lock.
If handler calls printf() → deadlock (same lock, non-recursive)!

Best practice: Set a volatile sig_atomic_t flag in handler,
               check it in main loop.
```

```c
volatile sig_atomic_t got_signal = 0;

void handler(int sig) {
    got_signal = 1;  // Just set flag — safe!
}

int main() {
    signal(SIGTERM, handler);
    while (!got_signal) {
        // Normal processing
        do_work();
    }
    // Clean shutdown
    cleanup();
    return 0;
}
```

### Signal Masks — Blocking Signals

```c
sigset_t mask, old_mask;

// Block SIGINT temporarily
sigemptyset(&mask);
sigaddset(&mask, SIGINT);
sigprocmask(SIG_BLOCK, &mask, &old_mask);

// Critical section — SIGINT won't interrupt this
do_critical_work();

// Unblock — if SIGINT arrived, it's delivered NOW
sigprocmask(SIG_SETMASK, &old_mask, NULL);
```

### Signal Delivery to Threads

```
Process-level signals (SIGTERM, SIGINT):
  Delivered to ANY thread that hasn't blocked it.
  Usually the main thread.

Thread-level signals (SIGSEGV, SIGFPE):
  Delivered to the thread that CAUSED the signal.

Best practice for multi-threaded programs:
  1. Block all signals in all threads except one
  2. Dedicated signal-handling thread uses sigwait()
  
void* signal_thread(void* arg) {
    sigset_t wait_set;
    sigemptyset(&wait_set);
    sigaddset(&wait_set, SIGTERM);
    sigaddset(&wait_set, SIGINT);
    
    int sig;
    sigwait(&wait_set, &sig);  // Blocks until signal arrives
    printf("Received signal %d\n", sig);
    // Clean shutdown
    return NULL;
}
```

### Signals and fork/exec

```
fork():
  - Signal handlers are INHERITED by child
  - Pending signals are NOT inherited
  - Signal mask IS inherited

exec():
  - Custom handlers reset to SIG_DFL (code no longer exists!)
  - Ignored signals (SIG_IGN) stay ignored
  - Signal mask IS preserved
  - Pending signals stay pending
```

---

## 13.3 The `self-pipe` Trick

Problem: `select()`/`poll()` can only wait on file descriptors, not signals.

```c
int pipe_fds[2];
pipe(pipe_fds);  // Create self-pipe

void signal_handler(int sig) {
    write(pipe_fds[1], "x", 1);  // Write to pipe (async-signal-safe!)
}

// In main event loop:
fd_set readfds;
FD_SET(pipe_fds[0], &readfds);
FD_SET(client_socket, &readfds);

select(max_fd + 1, &readfds, NULL, NULL, NULL);

if (FD_ISSET(pipe_fds[0], &readfds)) {
    // Signal arrived! Handle it safely here (outside handler)
    char buf;
    read(pipe_fds[0], &buf, 1);
    do_graceful_shutdown();
}
```

Modern alternative: `signalfd()` (Linux) — creates a file descriptor for signals.

---

## 13.4 Socket Programming — OS Networking Interface

### TCP Socket Lifecycle

```
Server:                              Client:
┌──────────────┐                     
│  socket()    │ Create socket       ┌──────────────┐
└──────┬───────┘                     │  socket()    │
       │                             └──────┬───────┘
┌──────┴───────┐                            │
│  bind()      │ Assign address             │
└──────┬───────┘                            │
       │                                    │
┌──────┴───────┐                            │
│  listen()    │ Mark as passive             │
└──────┬───────┘                            │
       │                                    │
┌──────┴───────┐        3-way handshake     │
│  accept()    │←───────────────────────────┤ connect()
│  (BLOCKS)    │  SYN ──────→               │
│              │  ←────── SYN+ACK           │
│              │  ACK ──────→               │
└──────┬───────┘ Returns new socket         │
       │         (for this client)          │
       │                                    │
┌──────┴───────┐                     ┌──────┴───────┐
│  read/write  │←───── data ────────→│  read/write  │
└──────┬───────┘                     └──────┬───────┘
       │                                    │
┌──────┴───────┐                     ┌──────┴───────┐
│  close()     │  FIN ←────→ FIN     │  close()     │
└──────────────┘                     └──────────────┘
```

### TCP Server Code

```c
#include <sys/socket.h>
#include <netinet/in.h>

int main() {
    // 1. Create socket
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    
    // 2. Set options (allow port reuse)
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    
    // 3. Bind to address
    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_addr.s_addr = INADDR_ANY,  // Listen on all interfaces
        .sin_port = htons(8080)          // Port 8080
    };
    bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
    
    // 4. Listen (backlog = max pending connections)
    listen(server_fd, 128);
    
    // 5. Accept connections
    while (1) {
        struct sockaddr_in client_addr;
        socklen_t len = sizeof(client_addr);
        int client_fd = accept(server_fd, 
                               (struct sockaddr*)&client_addr, &len);
        
        // Handle client (in real server: fork/thread/event loop)
        char buf[1024];
        int n = read(client_fd, buf, sizeof(buf));
        write(client_fd, "HTTP/1.1 200 OK\r\n\r\nHello!\n", 26);
        close(client_fd);
    }
}
```

### Server Architectures

```
1. FORK PER CONNECTION:
   while (1) {
       client = accept(server);
       if (fork() == 0) {
           handle(client); exit(0);
       }
       close(client);  // parent closes its copy
   }
   ✅ Simple, isolated
   ❌ Slow (fork overhead), max ~1000 connections

2. THREAD PER CONNECTION:
   while (1) {
       client = accept(server);
       pthread_create(&t, NULL, handle, client);
   }
   ✅ Faster than fork
   ❌ Thread overhead, max ~10,000 connections

3. THREAD POOL:
   Create N threads upfront.
   Dispatch incoming connections to available threads.
   ✅ Bounded resource usage
   ❌ Still limited by thread count

4. EVENT-DRIVEN (epoll):  ← HOW GOOGLE DOES IT
   while (1) {
       events = epoll_wait(epfd, ...);
       for each ready event:
           handle non-blocking I/O
   }
   ✅ Single thread, millions of connections (C10K+)
   ❌ Complex programming model (callbacks/state machines)

5. HYBRID (thread pool + epoll):
   Multiple threads, each running epoll.
   ✅ Best of both worlds
   ❌ Most complex
   Used by: nginx, Google's servers
```

---

## 13.5 TCP/IP Stack in the Kernel

### Packet Reception Path

```
Network Wire
     │
     ▼
┌──────────────┐
│     NIC      │ DMA transfer to ring buffer in RAM
│              │ Raise hardware interrupt (or NAPI poll)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Driver     │ Allocate sk_buff, fill metadata
│ (Interrupt   │ Pass up to network stack
│  handler)    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Network Layer│ Check IP header, routing decision
│ (IP)         │ Reassemble fragments if needed
│              │ Netfilter hooks (iptables/nftables)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Transport    │ Demultiplex by port number
│ Layer (TCP)  │ TCP state machine processing
│              │ ACK handling, congestion control
│              │ Place data in socket receive buffer
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Socket Layer │ Wake up process blocked in read()/recv()
│              │ Or: make epoll_wait() return this fd as ready
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Application  │ read(socket_fd, buf, n)
│              │ Data copied from kernel buffer to user buffer
└──────────────┘
```

### TCP State Machine

```
This is the FULL TCP state diagram — know it cold!

                              CLOSED
                                │
                    ┌───────────┼───────────┐
                    │ (passive open)    (active open)
                    ▼                       ▼
                 LISTEN                  SYN_SENT
                    │                       │
              (recv SYN,              (recv SYN+ACK,
               send SYN+ACK)          send ACK)
                    │                       │
                    ▼                       ▼
                SYN_RCVD ──────────→ ESTABLISHED
                                   (data transfer)
                                        │
                    ┌───────────────────┤
                    │              (recv FIN,
              (send FIN)           send ACK)
                    │                   │
                    ▼                   ▼
                FIN_WAIT_1         CLOSE_WAIT
                    │                   │
              (recv ACK)          (send FIN)
                    │                   │
                    ▼                   ▼
                FIN_WAIT_2          LAST_ACK
                    │                   │
              (recv FIN,          (recv ACK)
               send ACK)               │
                    │                   ▼
                    ▼               CLOSED
                TIME_WAIT
              (wait 2×MSL)
                    │
                    ▼
                 CLOSED
```

### TIME_WAIT — Why Wait 2×MSL?

```
After closing a connection, the socket stays in TIME_WAIT for 
2 × MSL (Maximum Segment Lifetime, typically 60 seconds).

Why?
  1. Ensure the final ACK reaches the remote end
     (if it's lost, remote will retransmit FIN, and we need to re-ACK)
  
  2. Allow old duplicate packets to expire
     (if a new connection uses same port, old packets could be
      misinterpreted as belonging to the new connection)

Problem: High-traffic server with many short connections
  → thousands of sockets in TIME_WAIT
  → can run out of port numbers!

Solutions:
  - SO_REUSEADDR: allow binding to TIME_WAIT address
  - tcp_tw_reuse: kernel reuses TIME_WAIT sockets for outgoing
  - Connection pooling (avoid creating/destroying connections)
```

---

## 13.6 `epoll` Deep Dive

### Level-Triggered vs Edge-Triggered

```
Level-Triggered (default):
  "Tell me whenever the fd IS ready"
  epoll_wait returns as long as data is available
  
  Safe, simple, but can cause redundant wakeups.
  
Edge-Triggered (EPOLLET):
  "Tell me when the fd BECOMES ready"
  epoll_wait returns ONCE when new data arrives
  
  More efficient, but you MUST read ALL available data!
  If you don't read everything, you won't be notified again.

Edge-triggered code pattern:
  while (1) {
      int n = read(fd, buf, sizeof(buf));
      if (n == -1 && errno == EAGAIN)
          break;  // No more data — done for now
      process(buf, n);
  }
```

### epoll Internals

```
Kernel maintains:
  1. Red-black tree of monitored file descriptors
     (for O(log n) add/remove)
  
  2. Ready list (linked list of fds with events)
     (returned to user on epoll_wait)

  3. Callback functions registered with each fd
     (when fd becomes ready, callback adds it to ready list)

This is why epoll is O(1) per event:
  - Kernel doesn't scan all fds
  - Ready fds add themselves to the ready list via callbacks
  - epoll_wait just returns the ready list
```

---

## 13.7 Zero-Copy Networking

### The Problem

```
Normal read + send:
  Disk → Kernel buffer → User buffer → Kernel buffer → NIC
         (1st copy)      (2nd copy)    (3rd copy)

4 context switches, 3 data copies for one file transfer!
```

### sendfile() — Avoid User-Space Copy

```c
sendfile(client_fd, file_fd, &offset, count);

// Disk → Kernel buffer → NIC buffer
// Only 2 copies, 2 context switches!
// Data never enters user space.

// With DMA gather:
// Disk → Kernel buffer ──(DMA)──→ NIC
// Only 1 copy!
```

### splice() — Zero-Copy Between Pipes/Sockets

```c
// Move data between two fds through a pipe (zero-copy in kernel)
int pipe_fds[2];
pipe(pipe_fds);
splice(file_fd, NULL, pipe_fds[1], NULL, count, 0);  // File → pipe
splice(pipe_fds[0], NULL, socket_fd, NULL, count, 0); // Pipe → socket
```

---

## 📝 Interview Questions — Test Yourself

### Signals
1. What happens when you send SIGKILL to a zombie process?
2. Why is `printf()` unsafe in a signal handler?
3. Explain the self-pipe trick. When would you use it?
4. What happens to signal handlers after `fork()`? After `exec()`?
5. How do you handle signals in a multi-threaded program?

### Networking
6. Explain the TCP 3-way handshake. Why three steps, not two?
7. What is TIME_WAIT? Why does it exist? How do you handle too many TIME_WAITs?
8. Compare level-triggered vs edge-triggered epoll. Trade-offs?
9. What is the C10K problem? How did epoll solve it?
10. Explain zero-copy networking (sendfile). When is it useful?

### Design
11. Design a high-performance HTTP server. What architecture would you use?
12. How does a load balancer work at the OS level? (L4 vs L7)

### Answers Guide

<details>
<summary>Click to reveal answers</summary>

**1.** Nothing useful. A zombie is already dead — it has no code running. SIGKILL can't kill it because there's nothing to kill. The only way to remove a zombie is for its parent to call `wait()` (or for the parent to die, letting init adopt and reap the zombie).

**2.** `printf()` uses internal locks (mutex) for thread safety and internal buffers. If a signal arrives while `printf()` holds its lock, and the signal handler also calls `printf()`, the handler will try to acquire the same lock → **deadlock**. Additionally, `printf()` uses global state (file buffers) that could be in an inconsistent state when the signal arrives.

**6.** Three-way handshake (SYN, SYN-ACK, ACK):
- Two steps aren't enough because TCP connections are **bidirectional** — each direction needs its own sequence number synchronized.
- Step 1 (SYN): Client tells server its initial sequence number.
- Step 2 (SYN-ACK): Server acknowledges client's ISN and tells client its own ISN.
- Step 3 (ACK): Client acknowledges server's ISN. Now both sides know both ISNs.
- With only 2 steps, the server wouldn't know if the client received its ISN.

**9.** C10K = handling 10,000 concurrent connections on a single server.
- **Problem with select/poll**: Both scan ALL file descriptors on every call → O(n) per call. At 10K fds, this becomes the bottleneck.
- **epoll solution**: Kernel tracks ready fds via callbacks. `epoll_wait()` returns ONLY ready fds → O(1) per event, O(k) per call where k = ready count. Can handle millions of connections.
- **io_uring** takes this further: shared ring buffer eliminates syscall overhead entirely.

**11.** High-performance HTTP server design:
- **Architecture**: Multi-threaded event loop (one epoll loop per core)
- **Connections**: Non-blocking sockets, edge-triggered epoll
- **Parsing**: Zero-copy HTTP parsing (parse in-place in receive buffer)
- **Static files**: sendfile() for zero-copy file serving
- **Keep-alive**: Connection pooling, reuse connections
- **Timeouts**: Timer wheel for connection timeouts
- **Load balancing**: SO_REUSEPORT to distribute across threads
- Example: nginx uses exactly this architecture

</details>

---

*Next: [Module 14: Practice Problem Sets →](../14-Practice-Problems/README.md)*
