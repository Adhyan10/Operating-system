# Module 7: File Systems

## 🧠 What You'll Learn
- File system interface and operations
- Directory structures and path resolution
- Disk allocation methods (contiguous, linked, indexed)
- Inodes and the Unix file system model
- Journaling and crash recovery
- Virtual File System (VFS) layer
- Modern file systems: ext4, Btrfs, ZFS

---

## 7.1 What is a File System?

A file system provides **persistent, named, organized storage** on top of raw disk blocks.

```
User's view:          File system's job:        Disk reality:
/home/user/           Map names → blocks        ┌─────┐
├── docs/             Track free space          │Block│
│   ├── resume.pdf    Handle permissions        │  0  │
│   └── notes.txt     Ensure consistency        ├─────┤
├── code/             after crashes             │Block│
│   └── main.c                                 │  1  │
└── photos/                                    ├─────┤
    └── cat.jpg                                │ ... │
                                               └─────┘
```

### File Metadata (Attributes)

| Attribute | Description |
|-----------|-------------|
| **Name** | Human-readable identifier |
| **Type** | Regular file, directory, symlink, device, socket, pipe |
| **Size** | In bytes |
| **Permissions** | rwx for owner, group, others |
| **Owner/Group** | UID and GID |
| **Timestamps** | Created, modified, accessed (ctime, mtime, atime) |
| **Link count** | Number of hard links |
| **Block pointers** | Which disk blocks contain the data |

---

## 7.2 Directory Implementation

A directory is just a special file that maps **names → inode numbers**.

```
Directory entry:
┌──────────────────┬────────────┐
│  Filename        │ Inode #    │
├──────────────────┼────────────┤
│  .               │  1234      │ (self)
│  ..              │  100       │ (parent)
│  resume.pdf      │  5678      │
│  notes.txt       │  5680      │
│  main.c          │  9012      │
└──────────────────┴────────────┘
```

### Path Resolution: `/home/user/docs/resume.pdf`

```
1. Start at root inode (inode 2 on ext4)
2. Read root directory → find "home" → inode 50
3. Read inode 50 (directory) → find "user" → inode 100
4. Read inode 100 (directory) → find "docs" → inode 1234
5. Read inode 1234 (directory) → find "resume.pdf" → inode 5678
6. Read inode 5678 → get file metadata and block pointers

That's 5+ disk reads for one file open!
(Cached by the dentry cache in practice)
```

### Hard Links vs Symbolic Links

```
Hard Link:                          Symbolic Link:
┌──────────┐  ┌──────────┐        ┌──────────┐
│ file1.txt│  │ file2.txt│        │ link.txt │
│ inode:500│  │ inode:500│        │ inode:600│
└─────┬────┘  └─────┬────┘        └─────┬────┘
      │             │                    │
      └──────┬──────┘                    ▼
             ▼                     ┌──────────┐
        ┌─────────┐               │ content: │
        │ inode   │               │"/path/to/│
        │  500    │               │ original"│
        │ data→...│               └──────────┘
        │ links=2 │                    │
        └─────────┘                    ▼
                                  ┌─────────┐
                                  │ inode   │
                                  │  500    │
                                  └─────────┘

Hard link: Same inode, link count increases, can't cross filesystems
Symlink: Separate inode, contains path, can cross filesystems, can break
```

---

## 7.3 Disk Allocation Methods

### Contiguous Allocation

```
File A: blocks 0-4        File B: blocks 6-8
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ A │ A │ A │ A │ A │   │ B │ B │ B │   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7   8   9

Directory: {filename, start_block, length}

✅ Fast sequential AND random access (start + offset)
✅ Simple
❌ External fragmentation (holes between files)
❌ Can't grow files easily
❌ Need to know file size in advance
```

### Linked Allocation

```
File A: start at block 0
┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
│data|→4 │   │        │   │        │   │        │
└───┬────┘   │        │   │        │   │        │
 0  │        │   1    │   │   2    │   │   3    │
    │        └────────┘   └────────┘   └────────┘
    ▼
┌────────┐   ┌────────┐   ┌────────┐
│data|→7 │   │        │   │        │
└───┬────┘   │   5    │   │   6    │
 4  │        └────────┘   └────────┘
    ▼
┌────────┐
│data|∅  │  (end of file)
└────────┘
 7

✅ No external fragmentation
✅ Files can grow easily
❌ Terrible random access (must traverse chain)
❌ One bad pointer → lose rest of file
❌ Space overhead (pointer in each block)
```

### FAT (File Allocation Table) — Improved Linked

```
Move all pointers into a separate table in memory:

FAT (in memory):              Disk blocks:
┌───┬──────┐                ┌──────────┐
│ 0 │  →4  │ ───────────→  │ A data   │ block 0
│ 1 │ free │                │          │ block 1
│ 2 │  →6  │                │ B data   │ block 2
│ 3 │ free │                │          │ block 3
│ 4 │  →7  │ ───────────→  │ A data   │ block 4
│ 5 │ free │                │          │ block 5
│ 6 │ EOF  │                │ B data   │ block 6
│ 7 │ EOF  │                │ A data   │ block 7
└───┴──────┘                └──────────┘

File A: 0 → 4 → 7 → EOF
File B: 2 → 6 → EOF

✅ Random access: follow chain in RAM (fast)
❌ FAT must be in memory (large disk = large FAT)
```

### Indexed Allocation (Unix Inodes) ⭐

```
Each file has an INDEX BLOCK containing all its block pointers:

Inode for file A:
┌─────────────────────┐
│ Metadata            │
│ (size, permissions) │
├─────────────────────┤
│ Direct block 0  → 8 │─────→ [data block 8]
│ Direct block 1  → 3 │─────→ [data block 3]
│ ...                  │
│ Direct block 11 → 20│─────→ [data block 20]
├─────────────────────┤
│ Single indirect → 50│──┐
├─────────────────────┤  │
│ Double indirect → 60│  │    ┌─────────────┐
├─────────────────────┤  └──→ │ 50: [21,22, │
│ Triple indirect → 70│      │  23,24,...]  │
└─────────────────────┘      │ 1024 entries │
                              └──────┬──────┘
                                     │
                                     ▼
                              [data blocks 21,22,23,...]
```

### Inode Block Addressing (ext4, 4KB blocks)

```
12 direct blocks:                12 × 4KB = 48KB
1 single indirect (1024 ptrs):   1024 × 4KB = 4MB
1 double indirect:               1024² × 4KB = 4GB
1 triple indirect:               1024³ × 4KB = 4TB

Total max file size: ~4TB (with 4-byte block pointers)
ext4 uses 48-bit block addresses → up to 16TB files, 1EB filesystem
```

### 🔥 Google Interview Insight
> **Q: Why do inodes use the multi-level indirect block scheme?**
> 
> A: It's optimized for the common case. Most files are **small** (<4KB). With 12 direct pointers, small files need NO indirect blocks → just 1 inode read. Medium files use single indirect (2 reads). Only huge files need triple indirect. This matches the file size distribution perfectly — fast for common sizes, capable of large sizes.

---

## 7.4 Free Space Management

### Bitmap

```
Each bit represents one block: 1=free, 0=used

Block bitmap:
┌────────────────────────────────┐
│ 0 1 1 0 0 1 1 1 0 0 1 0 ...  │
│ ↑       ↑                     │
│ used    used                  │
│   ↑ ↑     ↑ ↑ ↑     ↑        │
│   free    free       free     │
└────────────────────────────────┘

For 1TB disk with 4KB blocks:
  256M blocks → 256M bits → 32MB bitmap
  
Finding free block: scan bitmap → O(n/word_size)
(Can use hardware "find first set bit" instructions)
```

### Linked Free List

```
Superblock → [free block 5] → [free block 8] → [free block 12] → NULL

Allocate: take head of list — O(1)
Free: add to head of list — O(1)
Finding contiguous blocks: expensive (random order)
```

---

## 7.5 Journaling and Crash Recovery

### The Problem

```
Moving a file (mv /a/foo /b/foo) requires:
  1. Add entry "foo" to directory /b
  2. Remove entry "foo" from directory /a
  3. (Maybe update inode)

If crash between step 1 and 2:
  File appears in BOTH directories! (link count wrong)

If crash between step 2 and 3:
  File is in /b but metadata might be inconsistent
```

### Solution: Journaling (Write-Ahead Logging)

```
Before modifying the file system, write the PLAN to a journal:

Journal (separate area on disk):
┌──────────────────────────────────────┐
│ Transaction #42                       │
│ Operation: move /a/foo → /b/foo      │
│ Changes:                              │
│   1. Add dirent to /b: {foo, inode5} │
│   2. Remove dirent from /a: {foo}    │
│   3. Update inode5 timestamps        │
│ COMMIT                                │
└──────────────────────────────────────┘

Then: apply changes to actual filesystem

On crash recovery:
  - Scan journal for committed but not-applied transactions
  - Replay them (idempotent operations)
  - Discard uncommitted transactions
```

### Journaling Modes

| Mode | What's journaled | Performance | Safety |
|------|-----------------|-------------|--------|
| **Journal** | Metadata + Data | Slowest | Safest |
| **Ordered** (ext4 default) | Metadata only, data written first | Medium | Good |
| **Writeback** | Metadata only, data order not guaranteed | Fastest | Can have stale data |

---

## 7.6 Virtual File System (VFS)

Linux supports many file systems through a **common abstraction layer**:

```
Applications
     │
     │  POSIX API: open(), read(), write(), stat()
     ▼
┌─────────────────────────────────────────┐
│           Virtual File System (VFS)      │
│                                         │
│  Defines interfaces:                    │
│    struct inode_operations { ... };      │
│    struct file_operations { ... };       │
│    struct super_operations { ... };      │
│    struct dentry_operations { ... };     │
└────┬──────────┬──────────┬──────────┬───┘
     │          │          │          │
     ▼          ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│  ext4  │ │  XFS   │ │  NFS   │ │ procfs │
└────────┘ └────────┘ └────────┘ └────────┘

Each filesystem implements the VFS interfaces.
User programs don't care which filesystem is underneath.
```

### Key VFS Objects

| Object | Purpose |
|--------|---------|
| **Superblock** | Filesystem metadata (size, block count, free blocks) |
| **Inode** | File metadata (permissions, size, block pointers) |
| **Dentry** | Directory entry (name → inode mapping, cached in memory) |
| **File** | Open file instance (position, flags, operations) |

---

## 7.7 ext4 File System Layout

```
┌────────┬─────────────────────────────────────────┐
│  Boot  │              Block Group 0               │
│ Block  │                                         │
│        ├─────┬─────┬────────┬────────┬───────────┤
│        │Super│Group│ Block  │ Inode  │  Data     │
│        │Block│Desc.│Bitmap  │Bitmap  │ Blocks    │
│        │     │Table│        │+ Table │           │
└────────┴─────┴─────┴────────┴────────┴───────────┘
         │              Block Group 1               │
         ├─────┬─────┬────────┬────────┬───────────┤
         │Super│Group│ Block  │ Inode  │  Data     │
         │(bkp)│Desc.│Bitmap  │Bitmap  │ Blocks    │
         └─────┴─────┴────────┴────────┴───────────┘
         ...more block groups...
```

### ext4 Features

| Feature | Description |
|---------|-------------|
| **Extents** | Replaces indirect blocks: {start_block, length} — much better for large files |
| **Delayed allocation** | Don't allocate blocks until actual write to disk (better contiguity) |
| **Multiblock allocator** | Allocate multiple blocks at once (reduces fragmentation) |
| **Journal checksums** | Detect corruption in journal |
| **Max file size** | 16 TB |
| **Max filesystem** | 1 EB (exabyte) |

---

## 7.8 Log-Structured File Systems & Copy-on-Write

### Log-Structured (LFS)

```
Treat the ENTIRE disk as a circular log.
All writes (data + metadata) go sequentially to the end.

Write pattern:
───────────────────────────────────────→
│ D1 │ I1 │ D2 │ D3 │ I2 │ D4 │ I3 │
───────────────────────────────────────→
  data  inode data data inode data inode

✅ Sequential writes only → excellent for write-heavy workloads
✅ Natural journaling (log IS the filesystem)
❌ Random reads need indirection (inode map)
❌ Garbage collection needed (clean old segments)
```

### Copy-on-Write (Btrfs, ZFS)

```
Never overwrite data in place. Write new version to free space.

Before write:                    After write:
Root → Node A → Data X          Root' → Node A' → Data X'
                                          ↓
                                 Old Root → Old Node A → Old Data X
                                 (can still access! → snapshots!)

✅ Instant snapshots (just keep old root pointer)
✅ Data integrity (old data preserved until new data committed)
✅ Built-in checksumming
❌ Write amplification (updating one byte may rewrite path to root)
❌ Fragmentation over time
```

---

## 📝 Interview Questions — Test Yourself

### Conceptual
1. Compare contiguous, linked, and indexed allocation. Trade-offs?
2. What is an inode? What information does it contain?
3. Hard link vs symbolic link — when to use each?
4. Explain journaling in file systems. Why is "ordered" mode the default in ext4?
5. What is the VFS layer and why is it important?

### Problem Solving
6. Calculate the maximum file size for an inode with 12 direct, 1 single indirect, 1 double indirect, 1 triple indirect pointers (4KB blocks, 4-byte pointers).
7. How many disk reads are needed to read byte 50,000,000 of a file using ext4 extents vs traditional indirect blocks?

### Design
8. Design a file system for a flash-based (SSD) storage device. What's different from HDD?
9. How would you implement file system snapshots?
10. Design a distributed file system (like GFS/HDFS). What are the key components?

### Answers Guide

<details>
<summary>Click to reveal answers</summary>

**6.** 4KB blocks, 4-byte pointers → 1024 pointers per block.
- Direct: 12 × 4KB = 48KB
- Single indirect: 1024 × 4KB = 4MB
- Double indirect: 1024 × 1024 × 4KB = 4GB
- Triple indirect: 1024³ × 4KB = 4TB
- Total: ~4TB + 4GB + 4MB + 48KB ≈ 4TB

**8.** SSD file system considerations:
- **No seek time**: Random access is fast → no need to optimize for sequential
- **Wear leveling**: Spread writes evenly (flash cells have limited write cycles)
- **Write amplification**: SSDs write in pages (4KB) but erase in blocks (256KB). Overwriting triggers read-erase-write cycle.
- **TRIM/DISCARD**: Tell SSD which blocks are free so it can erase proactively
- **Log-structured** design works well (sequential writes, less write amplification)
- Example: F2FS (Flash-Friendly File System) designed by Samsung for Linux

**10.** Distributed file system (GFS model):
- **Master node**: Metadata server (file→chunk mapping, chunk locations)
- **Chunk servers**: Store actual data in large chunks (64MB)
- **Client library**: Contacts master for metadata, reads/writes directly to chunk servers
- **Replication**: Each chunk replicated 3x across different racks
- **Consistency**: Single master simplifies; appends are atomic
- **Fault tolerance**: Master has write-ahead log + checkpoints; chunk servers detected via heartbeats

</details>

---

*Next: [Module 8: I/O Systems →](../08-IO-Systems/README.md)*
