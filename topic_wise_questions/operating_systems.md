# Operating Systems — Senior Engineer Interview Questions & Model Answers

---

## 🔵 PROCESSES & THREADS

---

**Q1: What is the difference between a process and a thread?**

**Model Answer:**
A process is an independent program in execution — it has its own memory space, file descriptors, heap, stack, and OS resources. A thread is a unit of execution *within* a process. Threads share the same address space, heap, and open file descriptors, but each has its own stack and register state.

The key distinction is isolation vs. efficiency. Creating a process is expensive because the OS must duplicate the parent's address space (even with copy-on-write, the overhead is significant). Creating a thread is cheap because it shares the existing process's memory. Communication between processes (IPC) is harder and slower — it requires pipes, sockets, shared memory, or message queues. Threads communicate trivially through shared memory, but that also makes synchronization bugs (race conditions, deadlocks) much easier to introduce.

From a fault isolation standpoint, if a process crashes, it dies alone. If a thread crashes (e.g., a segfault), it brings down the entire process, including all sibling threads.

---

**Q2: What are the different states a process can be in, and what triggers transitions between them?**

**Model Answer:**
The classic process state model includes:

- **New** — the process is being created, PCB allocated, resources assigned.
- **Ready** — the process has everything it needs to run, it's just waiting for the CPU.
- **Running** — currently executing on a CPU core.
- **Blocked/Waiting** — waiting for some event: I/O completion, a lock, a signal, a timer. It voluntarily relinquishes the CPU.
- **Terminated/Zombie** — execution is complete, but the parent hasn't yet called `wait()`, so the PCB is still in memory for the exit status to be retrieved.

Transitions:
- **New → Ready**: OS finishes loading the process.
- **Ready → Running**: Scheduler dispatches the process (context switch in).
- **Running → Ready**: Preemption — a higher priority process arrives, or the time quantum expires.
- **Running → Blocked**: Process makes a blocking syscall (e.g., `read()` on a socket with no data).
- **Blocked → Ready**: The awaited event occurs (I/O interrupt fires, lock released).
- **Running → Terminated**: Process calls `exit()` or is killed.

A zombie process exists because the exit status is a resource — the parent needs to reap it. If the parent dies without calling `wait()`, the child is reparented to `init`/`systemd`, which reaps it.

---

**Q3: What is a context switch? What information must be saved and restored?**

**Model Answer:**
A context switch is the mechanism by which the CPU switches from executing one process (or thread) to another. It's a pure overhead operation — no user-useful work happens during a context switch.

The OS must save the current process's **context** into its Process Control Block (PCB):
- All CPU registers (general purpose, instruction pointer, stack pointer, base pointer)
- Program counter
- CPU status flags (condition codes)
- Memory management info (page table base register, TLB state — or the TLB must be flushed)
- I/O state (open files, pending signals)

Then it loads the new process's context from its PCB.

What makes context switches expensive:
1. The direct cost of saving/restoring registers.
2. **Cache pollution** — the incoming process's data is not in L1/L2 cache, so the CPU will suffer cache misses for a while.
3. **TLB flushing** — since virtual-to-physical mappings change, the TLB (Translation Lookaside Buffer) must be invalidated. Modern CPUs support ASIDs (Address Space Identifiers) to partially mitigate this.

Thread context switches within the same process are cheaper because memory mappings don't change — the TLB doesn't need to be flushed.

---

**Q4: What is a race condition? What OS primitives exist to prevent them?**

**Model Answer:**
A race condition occurs when the correctness of a program depends on the relative timing or interleaving of operations by multiple threads or processes. The outcome is non-deterministic and dependent on scheduling.

Classic example: two threads both read a counter value (say, 5), both increment it independently, and both write back 6. The counter should be 7 — we've lost an update.

OS primitives to prevent this:

- **Mutex (Mutual Exclusion Lock):** Only one thread can hold the lock at a time. Others block until it's released. Provides mutual exclusion. Risk: deadlock if acquired in inconsistent order.
- **Semaphore:** A generalized counter. A binary semaphore behaves like a mutex. A counting semaphore controls access to a pool of N resources (e.g., a connection pool). Can also be used for signaling between threads.
- **Spinlock:** Like a mutex, but the waiting thread busy-waits (spins in a loop) instead of sleeping. Good for very short critical sections on multicore systems. Terrible for single-core or long wait times — wastes CPU cycles.
- **Read-Write Lock (RWLock):** Allows multiple concurrent readers OR one exclusive writer. Great for read-heavy workloads.
- **Monitor:** A higher-level construct combining a mutex with condition variables. A thread can wait on a condition while inside a monitor, atomically releasing the lock.
- **Condition Variable:** Used with a mutex to let threads sleep until a condition is true (`wait`, `signal`, `broadcast`).
- **Atomic operations:** Hardware-level compare-and-swap (CAS), fetch-and-add — lock-free operations on single words.

---

**Q5: What is deadlock? What are the four necessary conditions for it to occur?**

**Model Answer:**
Deadlock is a state where a set of processes are each waiting for a resource held by another process in the set, and none can proceed.

The four **Coffman conditions** — all four must hold simultaneously for deadlock to occur:

1. **Mutual Exclusion** — At least one resource must be held in a non-shareable mode (only one process can use it at a time).
2. **Hold and Wait** — A process is holding at least one resource while waiting to acquire additional resources held by other processes.
3. **No Preemption** — Resources cannot be forcibly taken from a process; they must be released voluntarily.
4. **Circular Wait** — A circular chain of processes exists: P1 waits for P2, P2 waits for P3, ... Pn waits for P1.

**Strategies to deal with deadlock:**
- **Prevention:** Eliminate one of the four conditions. E.g., impose a global ordering on lock acquisition to break circular wait.
- **Avoidance:** Use the Banker's Algorithm — before granting a resource, check if the system will remain in a "safe state."
- **Detection + Recovery:** Allow deadlock to occur, run a detection algorithm periodically, then kill or roll back a process.
- **Ostrich Algorithm:** Ignore the problem (used in most general-purpose OSes — deadlocks are rare and recovery is too expensive to always handle).

---

**Q6: What is the difference between preemptive and non-preemptive scheduling?**

**Model Answer:**
In **non-preemptive (cooperative) scheduling**, once a process is given the CPU, it keeps it until it voluntarily yields — either by blocking on I/O, making a syscall, or explicitly yielding. The OS cannot forcibly remove the CPU from a running process. This was the model in early Windows (3.1) and classic Mac OS. The downside is that a misbehaving or CPU-bound process can starve all others.

In **preemptive scheduling**, the OS can forcibly remove the CPU from a running process. This is triggered by a hardware timer interrupt firing at the end of a time quantum. Modern OSes (Linux, Windows NT+, macOS) are preemptive. This guarantees CPU time to all processes and improves responsiveness — an interactive process won't be starved by a background computation.

Preemption introduces complexity: since a process can be interrupted at any time, shared data structures in the kernel must be protected from concurrent access (kernel preemption requires spinlocks and careful design). That's why early Unix kernels were non-preemptive at the kernel level even if preemptive at user level.

---

**Q7: Describe common CPU scheduling algorithms and their trade-offs.**

**Model Answer:**

- **FCFS (First-Come, First-Served):** Simple, non-preemptive. Suffers badly from the **convoy effect** — short jobs stuck behind a long one. Poor average wait time.

- **SJF (Shortest Job First):** Optimal in terms of average waiting time, but requires knowing burst time in advance (generally impossible in practice). Non-preemptive version can cause starvation of long jobs.

- **SRTF (Shortest Remaining Time First):** Preemptive SJF. Optimal average waiting time but still requires predicting burst times. High context switch overhead.

- **Round Robin (RR):** Each process gets a fixed time quantum. At expiry, it's preempted and moved to the back of the ready queue. Great for interactive systems. The quantum size matters: too small → too many context switches; too large → degrades toward FCFS.

- **Priority Scheduling:** Processes have priorities; highest priority runs first. Can be preemptive or not. Problem: **starvation** of low-priority processes. Fix: **aging** — gradually increase the priority of waiting processes.

- **Multilevel Queue Scheduling:** Different queues for different classes (foreground/interactive vs. background/batch), each with its own algorithm.

- **Multilevel Feedback Queue (MLFQ):** Processes can move between queues based on behavior. New jobs start in the highest-priority queue. If they use up their time quantum, they drop to a lower-priority queue. I/O-bound processes (which rarely use their full quantum) stay high. This approximates SJF without needing burst time predictions. Used by Linux's CFS-like predecessors and Windows.

- **CFS (Completely Fair Scheduler — Linux):** Not time-quantum based in the traditional sense. Tracks **virtual runtime** per process. Always picks the process with the smallest virtual runtime (using a red-black tree). Niceness values scale virtual runtime. Provides fairness across all processes.

---

## 🟠 MEMORY MANAGEMENT

---

**Q8: Explain virtual memory. Why does it exist and what problems does it solve?**

**Model Answer:**
Virtual memory is an abstraction that gives each process the illusion that it has its own large, contiguous, private address space, independent of the actual physical memory (RAM) installed.

**Problems it solves:**

1. **Isolation and protection:** Without virtual memory, processes share physical addresses and can overwrite each other's memory (or the kernel's). Virtual memory enforces that each process sees only its own mapped pages. Accessing another process's memory causes a fault.

2. **Running programs larger than RAM:** Not all of a program needs to be in RAM at once. Infrequently used pages can be on disk (swap). The OS transparently loads them on demand (**demand paging**).

3. **Simplifying linking and loading:** Every program can be compiled to start at address 0x400000. The MMU handles the translation to wherever it actually lives in physical memory.

4. **Sharing:** Multiple processes can map the same physical page (e.g., shared libraries like libc). Read-only pages are shared; writes trigger **copy-on-write**.

**How it works:** The CPU's MMU translates virtual addresses to physical addresses using **page tables**. Each process has its own page table. The TLB caches recent translations for speed. If a page isn't in RAM, a **page fault** fires, the OS finds the page (or fetches from swap), updates the page table, and restarts the instruction.

---

**Q9: What is a page fault? Distinguish between minor and major page faults.**

**Model Answer:**
A page fault is a hardware exception raised by the MMU when a process accesses a virtual address that is not currently mapped to a physical page in RAM (or the mapping has insufficient permissions).

The OS page fault handler runs, determines the cause, and either:
- Resolves it (allocates a page, loads data from disk, updates the page table, returns)
- Kills the process with SIGSEGV (if it's an invalid access)

**Minor page fault:** The page exists somewhere in physical memory — it's just not mapped into this process's page table yet. No disk I/O is required. Examples:
- A process accesses a lazily-allocated page for the first time (anonymous page — OS allocates a zero page).
- A shared library page is already in memory from another process; this process just needs a new mapping.

**Major page fault:** The page is not in physical memory — it must be fetched from disk (swap space or a memory-mapped file). This is slow (milliseconds vs. nanoseconds). Too many major page faults indicate memory pressure and cause **thrashing**.

**Thrashing:** The OS spends more time swapping pages in and out than doing useful work. The working set of all processes exceeds available RAM. Solution: reduce multiprogramming degree, add RAM, or use a smarter page replacement algorithm.

---

**Q10: Describe common page replacement algorithms.**

**Model Answer:**

- **OPT (Optimal / Bélády's Algorithm):** Replace the page that will not be used for the longest time in the future. Provably optimal in terms of page fault rate. Not implementable in practice (requires future knowledge). Used as a benchmark.

- **FIFO (First-In, First-Out):** Replace the oldest page. Simple. Suffers from **Bélády's anomaly** — counterintuitively, giving more frames can increase page faults with FIFO.

- **LRU (Least Recently Used):** Replace the page not used for the longest time. Approximates OPT well. Expensive to implement perfectly — requires tracking exact access times. Usually approximated.

- **Clock Algorithm (Second-Chance):** Approximates LRU cheaply. Pages form a circular list. Each page has a reference bit set when accessed. On replacement, scan clockwise: if reference bit is 1, clear it (give a second chance) and move on; if 0, replace it.

- **LFU (Least Frequently Used):** Replace the page with the lowest access count. Problem: old pages that were heavily used in the past but are no longer needed stick around.

- **NRU (Not Recently Used):** Classifies pages into four classes based on reference and dirty bits. Cheaply approximates LRU.

- **Working Set Model:** A process should only be in memory if its entire working set (the set of pages it's actively using) fits in RAM. If not, swap it out entirely.

Linux uses a variant of the clock algorithm with two LRU lists (active and inactive) and other heuristics.

---

**Q11: What is the difference between internal and external fragmentation?**

**Model Answer:**

**Internal fragmentation** occurs when allocated memory is *larger* than what was requested. The wasted space is *inside* the allocated block. Example: if the allocator only gives out blocks in powers of 2, a 33-byte request gets a 64-byte block, wasting 31 bytes. Fixed-size partition schemes suffer this.

**External fragmentation** occurs when there *is* enough total free memory to satisfy a request, but it's scattered in non-contiguous chunks. No single contiguous block is large enough. Variable-size partition schemes (like `malloc` in user space) suffer this over time.

**Solutions:**
- **Compaction:** Defragment memory by moving all allocated blocks together. Expensive and requires relocatable code.
- **Paging:** By using fixed-size pages, external fragmentation at the page level is eliminated — physical pages don't need to be contiguous. But internal fragmentation is introduced (partly empty pages).
- **Segmentation:** Allows variable-size segments but suffers external fragmentation.
- **Slab allocator (Linux kernel):** Pre-allocates caches of commonly used object sizes to minimize both types of fragmentation.

---

**Q12: What is copy-on-write (COW) and how does the OS use it?**

**Model Answer:**
Copy-on-write is a memory optimization where two parties sharing a resource only get their own private copy when one of them attempts to *modify* it. Until then, they share the same physical page, saving memory and time.

**Use in `fork()`:** When a process calls `fork()`, the naive approach would be to copy the entire parent address space into the child. This is expensive — the child might immediately call `exec()` and never use any of that copied data.

With COW, the OS marks all parent pages as **read-only** and shared between parent and child. Both share the same physical pages. When either one tries to write to a page, the MMU raises a protection fault. The OS handler sees it's a COW fault, allocates a new physical page, copies the content, updates the page table to point to the private copy, marks it writable, and resumes. Only modified pages are ever duplicated.

**Other uses of COW:**
- `mmap()` of files: writable file mappings are COW — modifications don't write back to the file until `msync()`.
- Snapshotting in databases and VMs.
- `vmsplice`, `sendfile`, and zero-copy I/O operations.
- String interning and shared memory in language runtimes.

---

**Q13: What is the TLB and why is it critical for performance?**

**Model Answer:**
The TLB (Translation Lookaside Buffer) is a small, extremely fast hardware cache inside the CPU (typically 32–1024 entries) that stores recent virtual-to-physical page translations.

Without the TLB, every memory access would require a **page table walk** — multiple memory accesses to traverse the multi-level page table hierarchy (on x86-64 it's 4 levels). That would mean every single memory access triggers 4 additional memory reads. Performance would be catastrophic.

The TLB caches the result of recent translations. On a **TLB hit**, translation is done in a single cycle. On a **TLB miss**, the CPU (hardware page table walker on x86) or OS (software TLB on MIPS) must walk the page table and load the result into the TLB.

**TLB shootdown:** When a process's page table is modified (e.g., page unmapped), other CPU cores caching that translation must invalidate their TLB entries. The OS sends an inter-processor interrupt (IPI) to all CPUs to flush TLB entries for the affected address range. This is expensive on multicore systems with many threads.

**Context switch cost:** On a context switch to a different process, the TLB must be flushed (since virtual addresses mean different things for different processes). Modern CPUs support **ASIDs (Address Space Identifiers)** — each TLB entry is tagged with an ASID, so entries from different processes can coexist, reducing flush frequency.

---

## 🟢 FILE SYSTEMS

---

**Q14: How does a traditional Unix file system (like ext4) work? What is an inode?**

**Model Answer:**
A Unix file system organizes data into:

- **Superblock:** Contains metadata about the file system — total blocks, free blocks, inode count, block size, mount state. It's the first thing the OS reads when mounting.
- **Inode table:** An array of inodes. Each file/directory has exactly one inode.
- **Data blocks:** Where actual file contents and directory entries live.
- **Block bitmap / inode bitmap:** Tracks which blocks/inodes are free.

**Inode:** A data structure that stores all metadata about a file *except its name*. It contains:
- File type (regular, directory, symlink, device, etc.)
- Permissions (owner, group, world r/w/x)
- Owner UID and GID
- Timestamps: atime (last access), mtime (last modification), ctime (last inode change)
- File size
- Hard link count
- Pointers to data blocks: direct pointers, single indirect (pointer to block of pointers), double indirect, triple indirect.

**Why is the name not in the inode?** Because a file can have multiple hard links — multiple directory entries pointing to the same inode. The name lives in the directory, which maps filenames to inode numbers.

**Directory:** A special file whose content is a list of (filename, inode number) pairs. When you open `/home/user/file.txt`, the OS walks the path: resolves `/` (inode 2), reads its directory, finds `home` → inode N, reads inode N, finds `user` → inode M, reads inode M, finds `file.txt` → inode K, reads inode K to get the file's metadata and data block pointers.

---

**Q15: What is journaling in a file system? Why does it matter?**

**Model Answer:**
Without journaling, a file system operation like "create a file" involves multiple separate disk writes: update the inode, update the directory, update the inode bitmap, update the block bitmap. If the system crashes between any two of these writes, the file system is in an **inconsistent state** — an inode might claim a block that the bitmap says is free, or vice versa.

Traditionally, `fsck` (file system check) had to scan the *entire* disk on reboot to find and repair inconsistencies. On large disks, this could take hours.

**Journaling** solves this with a **write-ahead log (journal)**. Before actually modifying any file system structures, the OS writes a description of the intended changes to the journal. Once the journal entry is committed (flushed to disk), the actual modifications are applied. After they complete, the journal entry is cleared.

If a crash occurs:
- Before the journal commit: the operation is simply lost (file was never created). Consistent.
- After the journal commit but before metadata updates: on replay, the OS re-applies the journal entry. Consistent.
- After everything: journal entry is already cleared. Consistent.

**Journaling modes (ext3/ext4):**
- **Writeback:** Only metadata is journaled. File data may be written before or after the journal commit. Fast but can expose stale data.
- **Ordered (default):** Metadata is journaled; data is written to disk *before* the journal commit. Good balance.
- **Full (data journaling):** Both data and metadata are journaled. Slowest but safest.

Other approaches: **log-structured file systems** (LFS), **copy-on-write file systems** (ZFS, btrfs — never overwrite in place, always write new versions, inherently consistent).

---

**Q16: What is the difference between a hard link and a symbolic link?**

**Model Answer:**

**Hard link:** A directory entry that directly references an inode. Multiple hard links to the same inode mean the file has multiple names — they are all equal. The file's data persists as long as any hard link exists (inode's link count > 0). Deleting one hard link just decrements the count; only when it hits 0 is the inode and data freed.

Limitations of hard links:
- Cannot cross file system boundaries (inodes are only unique within a file system).
- Cannot link to directories (would create cycles in the directory tree, breaking tools like `find` and `du`).

**Symbolic link (symlink):** A special file type whose content is a *path string* pointing to another file or directory. The OS follows the path at the time of access — it's a redirection. It has its own inode.

Symbolic links:
- Can cross file system boundaries.
- Can point to directories.
- Can be dangling (pointing to a nonexistent target).
- Add one level of indirection (path resolution).

In practice: if you `rm` the target of a symlink, the symlink becomes broken (dangling). If you `rm` a hard link, the other links still work because the underlying inode/data is unaffected.

---

**Q17: What is the VFS (Virtual File System) layer and why does it exist?**

**Model Answer:**
VFS (Virtual File System) is an abstraction layer in the kernel that provides a **uniform interface** to all file systems. It allows user-space programs to use the same system calls (`open`, `read`, `write`, `close`, `stat`) regardless of whether the underlying file system is ext4, XFS, NTFS, FAT32, NFS, procfs, tmpfs, or a FUSE file system.

Without VFS, every application would need to know the specifics of each file system. Instead, VFS defines a set of abstract objects:
- **Superblock object:** Represents a mounted file system.
- **Inode object:** Represents a file or directory.
- **Dentry (directory entry) object:** Represents a name-to-inode mapping, cached for path lookup speed.
- **File object:** Represents an open file from the perspective of a process.

Each concrete file system registers implementations of these abstract operations (function pointers in C). The kernel calls the abstract operations; the specific file system provides the implementation.

This also enables **stacking and composition**: a network file system (NFS) looks just like a local file system to applications. `procfs` and `sysfs` expose kernel data structures as files without storing anything on disk — they implement the VFS interface but generate content on the fly.

---

## 🔴 I/O & STORAGE

---

**Q18: What are the different I/O models? Explain blocking, non-blocking, synchronous, asynchronous I/O.**

**Model Answer:**
These two axes (blocking/non-blocking and synchronous/asynchronous) are often confused.

**Blocking I/O:** The calling thread suspends until the operation completes. Simple to program, wastes thread time waiting.

**Non-blocking I/O:** The system call returns immediately. If data isn't ready, it returns an error (`EAGAIN`/`EWOULDBLOCK`). The process must poll or use `select`/`poll`/`epoll` to know when the resource is ready. More complex but allows a single thread to manage many I/O operations.

**Synchronous I/O:** The process is responsible for initiating and completing the I/O. The process drives the transfer. Includes blocking and non-blocking I/O — in both cases, the *process* does the work.

**Asynchronous I/O (AIO):** The process submits an I/O request and immediately continues executing. The OS (or hardware) completes the I/O in the background and notifies the process when done (via a callback, signal, or completion queue). The process is *not* blocked. Linux's `io_uring` is the modern, high-performance AIO interface — it uses shared ring buffers between user space and kernel, eliminating syscall overhead per I/O.

**`select`/`poll`/`epoll`:**
- `select`: Monitors a set of FDs. O(n) scan, limited to 1024 FDs. Portable.
- `poll`: Similar to select but no FD limit. Still O(n).
- `epoll` (Linux): Event-driven, O(1) per event notification. Stores interest list in kernel; only returns ready FDs. Scales to millions of connections — the foundation of Node.js, nginx, etc.

---

**Q19: How does DMA (Direct Memory Access) work and why is it important?**

**Model Answer:**
Without DMA, the CPU must be involved in every byte transfer between a device and RAM: read a byte from the device's I/O port, write it to a memory address, repeat. This is called **programmed I/O (PIO)**. For large transfers (disk reads, network packets), this wastes enormous CPU time doing nothing but copying data.

**DMA** offloads this work to a dedicated DMA controller (or the device itself in modern "bus-mastering" devices). The CPU programs the DMA controller with:
- Source address (device or memory)
- Destination address (memory)
- Transfer size

Then the CPU is free to do other work. The DMA controller performs the transfer autonomously over the system bus, copying data directly between the device and main memory without touching the CPU. When done, it raises an **interrupt** to notify the CPU.

**Why it matters:** A disk read of 4KB would require 4096 individual CPU operations without DMA. With DMA, it's one setup operation and one interrupt. CPU utilization drops dramatically for I/O-heavy workloads.

**Cache coherency concern:** DMA writes directly to physical memory. The CPU may have a cached (stale) view of that memory region. The OS must either invalidate cache lines in the affected region before a DMA read, or flush them before a DMA write. Modern hardware provides cache-coherent DMA so software doesn't have to manage this explicitly.

---

**Q20: What is the difference between buffered and unbuffered I/O?**

**Model Answer:**
**Buffered I/O** (the default via C's `stdio` library — `fread`, `fwrite`) maintains an in-process buffer. Small writes are accumulated in user space and flushed to the OS in larger chunks. Small reads pull ahead from the OS in chunks. This dramatically reduces the number of system calls (which have overhead) and improves throughput for small, frequent operations.

**Unbuffered (direct) I/O** calls the OS syscall for every operation (e.g., `read()`, `write()` directly). This gives more control — each `write()` goes directly to the kernel buffer cache. Used when you need deterministic timing, low latency, or when you're managing your own buffering.

**Kernel buffer cache (page cache):** Even "unbuffered" userspace I/O still goes through the kernel's page cache. Disk reads are cached in RAM; writes are batched and written back lazily by the `pdflush`/`writeback` kernel threads. This is why `write()` returns quickly even though the disk is slow — data sits in the page cache. `fsync()` forces a flush of the page cache to disk.

**O_DIRECT:** A flag that bypasses the kernel page cache entirely, going straight to the device. Used by databases (like PostgreSQL, MySQL InnoDB) that maintain their own buffer pools and don't want double-buffering. Requires aligned buffers and aligned sizes.

---

## 🟣 INTER-PROCESS COMMUNICATION (IPC)

---

**Q21: What IPC mechanisms does a Unix OS provide? Compare their trade-offs.**

**Model Answer:**

- **Pipes (unnamed):** A unidirectional byte stream between related processes (parent-child). Simple, but only between processes with a common ancestor that set it up. No persistence.

- **Named pipes (FIFOs):** Like pipes but accessed via a file system path, enabling unrelated processes to communicate. Still unidirectional, byte-stream.

- **Signals:** Asynchronous notifications delivered to a process. Very limited — only a small integer is conveyed. Used for lifecycle events (`SIGTERM`, `SIGKILL`, `SIGHUP`) and exceptional conditions (`SIGSEGV`, `SIGFPE`). Cannot carry data. The handler runs asynchronously and must be async-signal-safe.

- **Message queues:** A kernel-maintained queue of messages. Multiple producers and consumers. Messages have types, allowing selective receive. POSIX (`mq_open`) and SysV variants. Persist until explicitly deleted. Better than pipes for structured, typed messages.

- **Shared memory:** The fastest IPC. Two processes map the same physical pages into their address spaces. One writes, the other reads — no data copying, no kernel involvement in the transfer itself. Must be protected with semaphores or mutexes. No inherent synchronization.

- **Semaphores:** Used for synchronization between processes (not just threads). POSIX named semaphores or SysV semaphores.

- **Sockets (Unix domain sockets):** Full-duplex, bi-directional communication. Can be stream (`SOCK_STREAM`) or datagram (`SOCK_DGRAM`). Can transfer file descriptors between processes (`SCM_RIGHTS`). Same API as network sockets, making code portable. Widely used (Docker, systemd, DBus all use Unix sockets).

- **Memory-mapped files (`mmap`):** Multiple processes can map the same file into their address spaces. Changes to the mapped region can be shared. Extremely efficient for large data structures.

**Throughput ranking:** Shared memory > mmap > Unix sockets > pipes > message queues. Latency-wise, shared memory also wins. But shared memory requires the most careful synchronization.

---

## ⚪ KERNEL & SYSTEM CALLS

---

**Q22: What is the difference between user mode and kernel mode?**

**Model Answer:**
Modern CPUs support (at minimum) two privilege levels:

**User mode (ring 3 on x86):** Unprivileged. Applications run here. The CPU prevents direct access to hardware, kernel memory, and privileged instructions (like `hlt`, writing to control registers, modifying page tables). If a user-mode process tries to execute a privileged instruction, the CPU raises a **general protection fault** — the OS kills the process.

**Kernel mode (ring 0 on x86):** Fully privileged. The OS kernel runs here. All CPU instructions are available. The kernel can access all physical memory, all I/O ports, control hardware, modify page tables, etc.

The separation exists for **protection and stability**. A buggy or malicious user process cannot corrupt the kernel or other processes because the CPU hardware enforces the boundary.

**Crossing the boundary — system calls:** When a process needs a privileged service (open a file, allocate memory, send a packet), it issues a syscall. On x86-64, this uses the `syscall` instruction, which atomically:
1. Saves user-mode register state.
2. Switches to kernel stack.
3. Transitions to ring 0.
4. Jumps to the syscall handler.

The handler validates arguments, performs the operation in kernel mode, then returns to user mode via `sysret`. This transition has overhead (~100-1000ns), which is why reducing syscall frequency matters for high-performance software.

---

**Q23: What happens from the moment you press a key on the keyboard until the character appears on screen?**

**Model Answer:**
This is a great end-to-end question touching interrupts, drivers, kernel, and more:

1. **Hardware interrupt:** The keyboard controller detects a key press and raises a hardware interrupt (IRQ1 on legacy systems) on the CPU's interrupt line.

2. **CPU stops current work:** The CPU finishes its current instruction, saves register state, looks up the interrupt in the **IDT (Interrupt Descriptor Table)**, and jumps to the registered ISR (Interrupt Service Routine) in kernel mode.

3. **Keyboard driver ISR:** The ISR reads the **scan code** from the keyboard controller I/O port. It translates the scan code to a key code, then to an ASCII character (depending on locale/key map).

4. **Input subsystem:** On Linux, the key event is passed to the input subsystem, which creates an `input_event` struct and delivers it to registered handlers.

5. **TTY layer / line discipline:** If a terminal is active, the character goes to the TTY layer. If in canonical (cooked) mode, the line discipline buffers until Enter. In raw mode, it's passed immediately.

6. **Process wakeup:** The terminal emulator (or whatever process has the TTY open) is blocked in a `read()` syscall. The kernel marks it as **ready** and the scheduler adds it to the run queue.

7. **Process reads data:** The terminal emulator process runs, reads the character from the kernel buffer.

8. **Rendering:** The terminal emulator calls the appropriate display API (X11/Wayland/framebuffer) to render the character — looks up the glyph in a font, computes pixels.

9. **GPU/display pipeline:** The rendered pixels are composited by the display server. The compositor sends a frame to the GPU. The GPU outputs the frame to the display via the HDMI/DisplayPort controller.

10. **You see the character.**

---

**Q24: What is the difference between a monolithic kernel and a microkernel? What about hybrid kernels?**

**Model Answer:**

**Monolithic kernel:** All core OS services run in kernel space in a single large binary: memory management, process scheduling, file systems, device drivers, networking. They all share the same address space and can call each other directly. This makes it *fast* — no IPC overhead for communication between subsystems. But a bug in a device driver can crash the entire system. Linux is monolithic (with loadable modules).

**Microkernel:** Minimizes what runs in kernel space. The kernel provides only: process/thread management, basic IPC, and memory address space management. Everything else — file systems, device drivers, networking, even libc — runs as **user-space servers**. Subsystems communicate via message-passing IPC. This improves **fault isolation** (a crashing file server doesn't take down the kernel) and **security** (smaller attack surface). The downside is **performance** — every service call crosses user-kernel boundaries multiple times. Mach (used in early macOS/iOS), QNX, MINIX, and seL4 are microkernels. L4 microkernels are designed around high-performance IPC.

**Hybrid kernel:** Pragmatically combines elements of both. Critical, performance-sensitive paths run in kernel mode; others may be moved out. Windows NT, macOS XNU (which combines a Mach microkernel with BSD kernel code running in kernel space), and Haiku use hybrid approaches. The term is sometimes controversial — critics argue they're just monolithic kernels that borrowed some microkernel concepts.

**Exokernel / Unikernel:** More extreme designs. Exokernel exposes hardware directly to applications, letting them implement their own OS abstractions. Unikernels link application + OS library into a single binary running directly on bare metal or a hypervisor — used in high-security/performance VMs.

---

## 🔵 VIRTUALIZATION & CONTAINERS

---

**Q25: What is the difference between a virtual machine and a container?**

**Model Answer:**

**Virtual Machine:** Uses a **hypervisor** to virtualize physical hardware. Each VM runs its own complete OS kernel, drivers, and user space. The hypervisor multiplexes physical CPU, RAM, and I/O across VMs.

- **Type 1 (bare-metal) hypervisor:** Runs directly on hardware (VMware ESXi, KVM, Hyper-V, Xen). The hypervisor *is* the host OS.
- **Type 2 (hosted) hypervisor:** Runs inside a host OS (VMware Workstation, VirtualBox). More overhead.

VMs provide **strong isolation** — different kernels, different memory spaces, hardware-level boundary. A compromised guest cannot easily escape. But they're **heavyweight** — GB of disk per VM, second-scale startup, significant RAM overhead per VM.

**Container:** Uses OS-level virtualization. Containers share the host kernel. Isolation is achieved through two Linux kernel features:
- **Namespaces:** Partition kernel resources. A container gets its own view of the PID namespace (PID 1 inside the container), network namespace (own IP, routing table), mount namespace (own file system view), UTS namespace (own hostname), IPC namespace, user namespace.
- **Cgroups (Control Groups):** Limit resource consumption — CPU, memory, I/O, network bandwidth.

Containers are **lightweight** — they start in milliseconds, use less memory, share kernel and binaries via union file systems (OverlayFS). But isolation is weaker — the kernel is shared; a kernel exploit potentially affects all containers.

**In practice:** Production systems often combine both — containers run inside VMs (e.g., Kubernetes nodes are VMs; pods are containers). This gives the performance/density of containers with the isolation of VMs.

---

**Q26: What are Linux namespaces and cgroups? How do they enable containers?**

**Model Answer:**

**Namespaces** wrap global kernel resources in an abstraction so that processes in different namespaces see different instances of those resources. Linux has (as of recent kernels) 8 namespace types:

- **PID namespace:** Process trees are isolated. PID 1 inside the namespace is just a regular process from the host's perspective. Processes in different namespaces can have the same PID.
- **Network namespace:** Each namespace gets its own network interfaces, routing tables, firewall rules, sockets. `veth` pairs bridge namespaces.
- **Mount namespace:** Each process can have its own view of the file system hierarchy. Pivoting the root (`pivot_root`) gives a container its own `/`.
- **UTS namespace:** Own hostname and NIS domain name.
- **IPC namespace:** Isolates System V IPC and POSIX message queues.
- **User namespace:** Maps UID/GID inside the namespace to different UIDs on the host. Allows a process to be "root" inside the container without being root on the host. Key for rootless containers.
- **Cgroup namespace:** Isolates the view of cgroup hierarchies.
- **Time namespace:** (Recent) Allows different system time offsets per namespace.

**Cgroups (Control Groups):** A kernel mechanism to limit, account for, and isolate resource usage. Organized as a hierarchy. You can create a cgroup, assign processes to it, and set limits:
- `memory.limit_in_bytes`: OOM kill processes in this cgroup if they exceed the limit.
- `cpu.shares` / `cpu.quota`: Limit CPU time.
- `blkio` / `io`: Limit disk bandwidth.
- `pids.max`: Limit number of processes (prevents fork bombs).

**How Docker uses them:** `docker run` calls `clone()` with namespace flags to create a new set of namespaces. It then calls `cgroup` APIs to set resource limits. It uses OverlayFS to layer the container image. The result is a process that *thinks* it's on its own machine.

---

## 🟡 SIGNALS, INTERRUPTS, SYNCHRONIZATION

---

**Q27: What is the difference between an interrupt and a trap? What is a syscall in this context?**

**Model Answer:**

**Interrupt (hardware interrupt):** An asynchronous signal generated by external hardware — keyboard, NIC, disk controller, timer. The CPU stops its current work, saves state, and runs the registered interrupt handler (ISR). Interrupts can happen at any time, completely unrelated to what the CPU was executing.

**Trap (software interrupt / exception):** A synchronous event generated by the CPU itself in response to the currently executing instruction. Examples:
- Division by zero
- Page fault (accessing an unmapped address)
- Invalid opcode
- Overflow
- Breakpoint (`INT3` — used by debuggers)
- General protection fault (privilege violation)

Traps are *synchronous* — they occur at a specific, determined instruction.

**System call:** A deliberate, controlled trap from user mode to kernel mode. The process uses a specific instruction (`syscall` on x86-64, `svc` on ARM) that triggers a trap, transferring control to the kernel's syscall handler at a predefined entry point. The syscall number (in `rax`) determines which service is requested. The kernel validates, executes, then returns to user mode.

The key distinction: interrupts are **asynchronous** (come from outside the CPU), traps are **synchronous** (caused by the currently running instruction), and syscalls are a **controlled subtype of trap** used intentionally by software.

**Interrupt priority:** Hardware interrupts have priority levels. Higher-priority interrupts can preempt lower-priority ISRs. The timer interrupt often has the highest priority to ensure scheduling preemption can always occur.

---

**Q28: What is priority inversion? How can it be solved?**

**Model Answer:**
Priority inversion is a scenario where a high-priority task is effectively blocked by a low-priority task, often mediated by a medium-priority task.

**Classic scenario:**
- Task H (high priority) needs a mutex held by Task L (low priority).
- Task H blocks waiting for the mutex.
- Task M (medium priority) runs, preempting Task L.
- Task L can't run to release the mutex because M keeps preempting it.
- Result: H waits for M to finish, even though H has higher priority than M. The priority ordering is inverted.

This famously crashed NASA's Mars Pathfinder rover in 1997 — a high-priority bus scheduling task was blocked behind a low-priority meteorological task, and a medium-priority task kept the low-priority task from completing.

**Solutions:**

- **Priority Inheritance:** When Task H blocks on a mutex held by Task L, temporarily raise Task L's priority to match Task H's. L can now preempt M and complete, release the mutex, and have its priority restored. This is the most common fix. Used in POSIX (`PTHREAD_PRIO_INHERIT`).

- **Priority Ceiling Protocol:** Every mutex is assigned a priority ceiling equal to the highest priority of any task that may acquire it. When a task acquires the mutex, its priority is raised to the ceiling. Prevents inversion entirely, at the cost of potentially elevating priority unnecessarily.

- **Avoid shared mutexes between priority levels:** Architectural redesign. High and low priority tasks don't share locks. Communication goes through lock-free queues or message passing.

---

**Q29: What is the difference between a spin lock and a mutex? When would you choose each?**

**Model Answer:**

**Mutex:** When a thread fails to acquire a mutex, it is put to sleep — the OS context-switches it out, adds it to a wait queue, and schedules another thread. When the mutex is released, the OS wakes up a waiting thread. This involves two context switches (sleep + wake).

**Spinlock:** When a thread fails to acquire a spinlock, it busy-waits — it loops, repeatedly checking the lock until it's free. No context switch, no OS involvement. Wastes CPU cycles while spinning.

**When to use each:**

Use a **spinlock** when:
- You're in kernel code where sleeping isn't allowed (interrupt handler — you can't sleep in an ISR; preemption may be disabled).
- The critical section is **extremely short** (nanoseconds) — a few instructions.
- You're on a **multicore** system, so another CPU can actually release the lock while you spin. On a single-core system, spinning is always wrong — the thread holding the lock can't run while you spin.
- The overhead of context switching is *greater* than the expected wait time.

Use a **mutex** when:
- You're in user space.
- The critical section may take any non-trivial time.
- Lock contention is expected.
- You want to avoid burning CPU cycles.

**Adaptive mutexes (modern Linux, Java HotSpot):** Spin briefly first; if the lock isn't released quickly, sleep. This gets the best of both worlds for most workloads.

---

## 🔵 BOOT & INITIALIZATION

---

**Q30: Walk me through what happens when a Linux machine boots — from power-on to login prompt.**

**Model Answer:**

1. **Power-on / Reset:** CPU initializes to a known state. The instruction pointer is set to a fixed address in ROM/flash where firmware lives.

2. **BIOS / UEFI firmware:** POST (Power-On Self-Test) — checks RAM, CPU, devices. Enumerates hardware. UEFI is the modern replacement for BIOS — it understands GPT partition tables and has a built-in bootloader protocol.

3. **Bootloader (GRUB2):** UEFI loads the bootloader from the EFI System Partition. GRUB reads its config, presents a menu (if configured), and loads the Linux kernel image (`vmlinuz`) and the **initial RAM disk** (`initrd`/`initramfs`) into memory.

4. **Kernel decompression:** `vmlinuz` is a compressed kernel image. It decompresses itself into memory.

5. **Kernel initialization:** The kernel starts executing. Initializes the memory subsystem, sets up page tables, initializes the CPU, detects hardware (PCI bus enumeration), loads built-in drivers.

6. **initramfs:** A temporary in-memory root file system bundled with the kernel. It contains just enough to mount the real root: disk drivers, LVM/RAID modules, file system drivers. The kernel runs `/init` inside initramfs.

7. **`pivot_root`:** Once the real root file system is mounted, the kernel pivots to it, discarding initramfs.

8. **PID 1 — init / systemd:** The kernel starts the first user-space process (`/sbin/init` or `systemd`) with PID 1. All other processes are descendants of PID 1.

9. **systemd initialization:** systemd reads unit files. It starts services in dependency order (parallelized). Mounts file systems (`/etc/fstab`), sets hostname, starts networking, udev (device manager), logging, etc.

10. **Display manager / getty:** systemd starts a display manager (GDM, LightDM) for graphical login, or `getty` processes on virtual terminals for text login.

11. **Login prompt / PAM:** The user types credentials. `getty` calls `login`, which uses PAM (Pluggable Authentication Modules) to verify. On success, the user's shell is spawned with their environment.

---

That's **30 questions** across processes, threads, memory management, file systems, I/O, IPC, kernel internals, virtualization, synchronization, and boot — the full breadth of what a top-tier OS interview covers. A candidate who can answer all of these fluently, with the nuance shown, would stand out at any company.