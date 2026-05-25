# Module 9: Virtualization and Containers

## 🧠 What You'll Learn
- What virtualization is and why it matters
- Hypervisor types (Type 1 vs Type 2)
- Hardware-assisted virtualization (VT-x, EPT)
- Containers: namespaces, cgroups, and how Docker works
- VMs vs Containers — when to use each
- Google's Borg and Kubernetes

> **Google heavily uses containers at scale.** This module connects OS concepts to real-world infrastructure.

---

## 9.1 Virtualization Fundamentals

### What is Virtualization?

Run multiple **virtual machines** (VMs) on a single physical machine, each with its own OS.

```
Without virtualization:           With virtualization:
┌─────────────────┐              ┌──────┐ ┌──────┐ ┌──────┐
│   Application   │              │App A │ │App B │ │App C │
├─────────────────┤              ├──────┤ ├──────┤ ├──────┤
│  Operating Sys  │              │Linux │ │Win11 │ │Linux │
├─────────────────┤              │(VM1) │ │(VM2) │ │(VM3) │
│   Hardware      │              └──┬───┘ └──┬───┘ └──┬───┘
└─────────────────┘                 │        │        │
                                 ┌──┴────────┴────────┴──┐
  One OS,                        │     Hypervisor (VMM)   │
  wastes resources               ├────────────────────────┤
                                 │      Hardware          │
                                 └────────────────────────┘
                                 
                                 Multiple OSes, better utilization
```

### Why Virtualize?

| Benefit | Explanation |
|---------|-------------|
| **Server consolidation** | Run 10 VMs on 1 server instead of 10 servers |
| **Isolation** | VM crash doesn't affect other VMs |
| **Security** | Smaller attack surface per VM |
| **Portability** | Move VMs between physical machines |
| **Testing** | Run different OS/versions simultaneously |
| **Cloud computing** | Entire cloud infrastructure is built on VMs |

---

## 9.2 Hypervisor Types

### Type 1 (Bare-Metal) — Directly on hardware

```
┌────────────────────────────────────┐
│  VM1 (Linux)  │  VM2 (Windows)    │
├────────────────────────────────────┤
│         Hypervisor (Type 1)        │  ← Runs directly on hardware
│    (Xen, VMware ESXi, KVM*)       │
├────────────────────────────────────┤
│            Hardware                │
└────────────────────────────────────┘

*KVM is technically Type 1 — it turns Linux kernel into a hypervisor
```

### Type 2 (Hosted) — On top of a host OS

```
┌────────────────────────────────────┐
│  VM1 (Linux)  │  VM2 (Windows)    │
├────────────────────────────────────┤
│         Hypervisor (Type 2)        │  ← Runs as application
│   (VirtualBox, VMware Workstation) │
├────────────────────────────────────┤
│          Host OS (Linux/Mac/Win)   │
├────────────────────────────────────┤
│            Hardware                │
└────────────────────────────────────┘
```

---

## 9.3 Virtualization Challenges

### The Trap-and-Emulate Problem

```
Normal OS execution:
  User mode → privileged instruction → TRAP → kernel handles

Virtualized:
  Guest OS thinks it's in kernel mode, but it's actually in user mode!
  
  Guest kernel → privileged instruction → TRAP → hypervisor handles
  
  The hypervisor EMULATES the privileged instruction:
  - Guest does "write to page table" → hypervisor intercepts,
    maps to shadow page table
  - Guest does "disable interrupts" → hypervisor notes this
    but doesn't actually disable hardware interrupts
```

### Hardware-Assisted Virtualization (Intel VT-x / AMD-V)

```
CPU privilege levels without VT-x:     With VT-x:
┌─────────────────┐                   ┌─────────────────┐
│ Ring 3: User app │                   │ Ring 3: User app │
├─────────────────┤                   ├─────────────────┤
│ Ring 0: Guest OS │ ← Problem!       │ Ring 0: Guest OS │ ← Real ring 0
│  (demoted)       │                   ├─────────────────┤
├─────────────────┤                   │ VMX Root Mode    │ ← New mode!
│ Ring 0: VMM      │                   │ (Hypervisor)     │
└─────────────────┘                   └─────────────────┘

VT-x adds:
  VM Exit:  Guest → Hypervisor (on privileged ops)
  VM Entry: Hypervisor → Guest (resume execution)
  VMCS:     VM Control Structure (stores guest/host state)
```

### Memory Virtualization: EPT (Extended Page Tables)

```
Without EPT (Shadow Page Tables):
  Guest VA → Guest PA → Host PA (two translations, OS must maintain both)
  
  Guest page table:     Shadow page table:
  GVA → GPA             GVA → HPA (maintained by hypervisor)
  
  Every guest page table change → hypervisor must update shadow
  Very expensive!

With EPT (hardware support):
  Guest VA → Guest PA → Host PA (hardware does BOTH translations)
  
  Guest page table:     EPT (hardware):
  GVA → GPA             GPA → HPA
  
  Two-level translation in hardware!
  Guest can modify its page tables freely.
  Only TLB miss triggers EPT walk.
```

---

## 9.4 Containers — Lightweight Virtualization

Containers share the **host kernel** but isolate processes using kernel features.

```
VMs:                              Containers:
┌──────┐ ┌──────┐ ┌──────┐      ┌──────┐ ┌──────┐ ┌──────┐
│App A │ │App B │ │App C │      │App A │ │App B │ │App C │
├──────┤ ├──────┤ ├──────┤      ├──────┤ ├──────┤ ├──────┤
│Libs  │ │Libs  │ │Libs  │      │Libs  │ │Libs  │ │Libs  │
├──────┤ ├──────┤ ├──────┤      └──┬───┘ └──┬───┘ └──┬───┘
│Guest │ │Guest │ │Guest │         │        │        │
│ OS   │ │ OS   │ │ OS   │      ┌──┴────────┴────────┴──┐
└──┬───┘ └──┬───┘ └──┬───┘      │    Container Runtime   │
┌──┴────────┴────────┴──┐       │    (Docker, containerd)│
│      Hypervisor        │      ├────────────────────────┤
├────────────────────────┤      │     Host OS Kernel     │
│     Host OS / Hardware │      ├────────────────────────┤
└────────────────────────┘      │      Hardware          │
                                └────────────────────────┘
VM: Full OS per instance         Container: Shared kernel
~GB of overhead per VM           ~MB of overhead per container
Boot in minutes                  Start in milliseconds
```

### Linux Namespaces — What Containers Can See

Namespaces provide **isolation** — each container has its own view of:

| Namespace | Isolates | Effect |
|-----------|----------|--------|
| **PID** | Process IDs | Container sees its own PID 1, can't see host processes |
| **Network** | Network stack | Own IP address, ports, routing table |
| **Mount** | Filesystem | Own root filesystem, mounts |
| **UTS** | Hostname | Own hostname |
| **IPC** | IPC resources | Own message queues, semaphores |
| **User** | UID/GID | Root in container ≠ root on host |
| **Cgroup** | Cgroup view | Own view of resource limits |

```c
// Creating a namespace (simplified):
int flags = CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS;
clone(child_func, stack, flags, arg);

// Child process now lives in a new PID + Network + Mount namespace
// It sees itself as PID 1
// It has an empty network stack
// It has its own mount table
```

### Cgroups — Resource Limits

```
Control groups limit HOW MUCH of each resource a container can use:

/sys/fs/cgroup/
├── cpu/
│   └── container1/
│       ├── cpu.cfs_quota_us: 50000    ← Max 50% of one CPU
│       └── cpu.cfs_period_us: 100000
├── memory/
│   └── container1/
│       ├── memory.limit_in_bytes: 512M ← Max 512MB RAM
│       └── memory.oom_control: 0       ← Kill if exceeded
├── blkio/
│   └── container1/
│       └── blkio.throttle.write_bps_device: 10MB/s
└── pids/
    └── container1/
        └── pids.max: 100               ← Max 100 processes
```

### How Docker Works

```
docker run -it ubuntu bash

What happens:
1. Docker daemon receives request
2. Pulls image layers (if not cached)
3. Creates container filesystem (UnionFS/OverlayFS):
   ┌────────────────┐ ← Read-write layer (container changes)
   ├────────────────┤
   │ Layer 3: app   │ ← Read-only image layers
   │ Layer 2: deps  │
   │ Layer 1: ubuntu│
   └────────────────┘
4. Creates namespaces (PID, Network, Mount, ...)
5. Sets up cgroups (CPU, memory limits)
6. Sets up networking (virtual ethernet pair + bridge)
7. Runs /bin/bash as PID 1 in the container

Container is just a Linux PROCESS with:
  - Isolated namespaces
  - Resource limits (cgroups)
  - Layered filesystem
```

---

## 9.5 VMs vs Containers

| Aspect | VMs | Containers |
|--------|-----|-----------|
| **Isolation** | Strong (separate kernel) | Weaker (shared kernel) |
| **Security** | Better (hypervisor boundary) | Kernel vulnerability = escape |
| **Overhead** | High (~GB per VM) | Low (~MB per container) |
| **Startup** | Minutes | Milliseconds |
| **Density** | ~10-50 per host | ~100-1000 per host |
| **Portability** | Any OS | Same kernel family |
| **Use case** | Multi-tenant, different OSes | Microservices, CI/CD |

### 🔥 Google Interview Insight
> **Q: Google runs everything in containers. How does Borg/Kubernetes ensure isolation?**
> 
> A: Google uses a defense-in-depth approach:
> 1. **Namespaces + cgroups**: Basic isolation
> 2. **Seccomp-BPF**: Filter which syscalls a container can make
> 3. **AppArmor/SELinux**: Mandatory access control
> 4. **gVisor**: Google's container sandbox — a user-space kernel that intercepts syscalls (stronger isolation than raw containers, lighter than VMs)
> 5. **Microservices architecture**: Even if one container is compromised, blast radius is limited

---

## 9.6 Google's Infrastructure

### Borg → Kubernetes

```
Borg (Google internal, 2004):
  - Manages containers across entire Google fleet
  - Two types of workloads:
    - Production (high priority): serving, storage
    - Batch (low priority): MapReduce, analytics
  - Bin-packing: fit workloads efficiently on machines
  - Handles failures: restarts, reschedules

Kubernetes (2014, open-source):
  - Inspired by Borg
  - Core concepts:
    ┌─────────────────────────────────────┐
    │ Pod: smallest deployable unit       │
    │      (1+ containers sharing network)│
    ├─────────────────────────────────────┤
    │ Service: stable endpoint for pods   │
    ├─────────────────────────────────────┤
    │ Deployment: manages pod replicas    │
    ├─────────────────────────────────────┤
    │ Node: worker machine                │
    ├─────────────────────────────────────┤
    │ Control Plane: API server, scheduler│
    └─────────────────────────────────────┘
```

---

## 📝 Interview Questions — Test Yourself

### Conceptual
1. Compare Type 1 and Type 2 hypervisors. Give examples.
2. What is trap-and-emulate? Why is it important for virtualization?
3. How do EPT (Extended Page Tables) improve VM performance?
4. What are Linux namespaces? Name and describe at least 5.
5. How do cgroups work? What resources can they limit?

### Design
6. Design a container runtime from scratch. What kernel features do you need?
7. How would you live-migrate a VM from one host to another without downtime?
8. Compare gVisor, Firecracker, and traditional containers. When to use each?

### Answers Guide

<details>
<summary>Click to reveal answers</summary>

**7.** VM live migration:
1. **Pre-copy**: While VM runs, copy all memory pages to destination
2. **Iterative copy**: Re-copy pages that were modified during step 1 (dirty pages)
3. **Repeat**: Until dirty page rate is low enough
4. **Stop-and-copy**: Pause VM, copy remaining dirty pages, transfer CPU state (VMCS)
5. **Resume**: Start VM on destination
Total downtime: typically <100ms

Challenges:
- Memory-intensive workloads: too many dirty pages
- Network bandwidth
- Storage: needs shared storage (SAN/NFS) or storage migration

**8.** 
- **Traditional containers** (Docker): Best performance, weakest isolation. For trusted workloads.
- **gVisor**: Intercepts syscalls in user-space kernel. Better isolation, 5-15% overhead. For untrusted but not adversarial workloads.
- **Firecracker** (AWS): MicroVM — full VM with minimal hypervisor. Boots in <125ms. Strong isolation with near-container startup. For multi-tenant serverless (Lambda, Fargate).

</details>

---

*Next: [Module 10: Concurrency in Practice →](../10-Concurrency-In-Practice/README.md)*
