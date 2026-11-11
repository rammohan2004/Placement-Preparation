# 16. OS Interview Question Bank (company-wise)

> How to use: answer **out loud** in 1–2 minutes, then check the linked note. ⭐ = very frequently asked. Each answer pointer is note §section.
> For a mock interview: *"mock interview me on OS, <company> style"*.

---

## 1. Universal top 25 (every company)
| # | Question | Where |
|---|---|---|
| 1 ⭐ | Process vs thread: what's shared, what's private? | 03 §2.1–2.2 |
| 2 ⭐ | What happens during a context switch? Cost? | 02 §2.5 |
| 3 ⭐ | What is a system call? Walk through `write()`. | 01 §2.3 |
| 4 ⭐ | fork/exec/wait; zombie vs orphan | 02 §2.6–2.7 |
| 5 ⭐ | How many times is "x" printed? (fork puzzles) | 02 §3 |
| 6 ⭐ | Mutex vs semaphore vs spinlock | 05 §2.5–2.7 |
| 7 ⭐ | Four conditions for deadlock; how to prevent in real code | 07 §2.1, §2.6 |
| 8 ⭐ | Banker's algorithm: run it on a table | 07 §3 |
| 9 ⭐ | Virtual memory: why, how, what's a page fault | 09 §2.1–2.2 |
| 10 ⭐ | Page replacement FIFO/LRU/OPT trace; Belady's anomaly | 09 §3 |
| 11 | Paging vs segmentation; internal vs external fragmentation | 08 §2.2–2.4 |
| 12 | Scheduling algorithms; Gantt chart numerical | 04 §3 |
| 13 | Producer-consumer with semaphores; why the order of waits matters | 06 §2.1 |
| 14 | Readers-writers; who starves? | 06 §2.2 |
| 15 | Dining philosophers solutions | 06 §2.3 |
| 16 | Thrashing and working set | 09 §2.5 |
| 17 | Inode; hard vs soft link | 10 §2.2–2.3 |
| 18 | Condition variable: why the while loop? | 05 §2.8 |
| 19 | User mode vs kernel mode; trap vs interrupt | 01 §2.1–2.2 |
| 20 | Monolithic vs microkernel | 01 §2.6 |
| 21 | IPC mechanisms; fastest one | 12 §2.1 |
| 22 | Copy-on-write fork | 02 §2.6, COA 07 §2.10 |
| 23 | TLB and its role; what happens on a context switch to the TLB | COA 07, 02 §2.5 |
| 24 | Race condition, critical section requirements, Peterson | 05 §2.1–2.3 |
| 25 | `malloc` internals; stack vs heap | 08 §2.6–2.7 |

---

## 2. Amazon [AMZN] (SDE: concepts + practical Linux)
- ⭐ Process vs thread; when would you use multiple processes instead of threads? (isolation: a crash in one doesn't kill the others; Chrome tabs)
- ⭐ Explain deadlock with an example from code you've written; how would you detect it in production? (07 §2.6, gdb backtraces)
- ⭐ Implement a thread-safe bounded blocking queue (06 §2.1 code)
- Print numbers alternately with 2 threads / FizzBuzz multithreaded (06 §2.4)
- What happens when you type a command in the shell and press Enter? (fork, exec, wait, path lookup, dup2 for redirects)
- A server is slow / at 100 % CPU: how do you investigate? (13 §2.7)
- VM vs container; what are cgroups/namespaces? Why does Lambda use Firecracker? (15)
- What happens when a container exceeds its memory limit? (OOMKilled, 15)
- `write()` returned but the data was lost after a crash: why? (10 §2.7)
- epoll vs select; how does a web server handle 10k connections? (11 §2.6)
- Design question flavour: how would you rate-limit / schedule jobs fairly? (scheduling ideas: RR, weighted fair share)

## 3. D. E. Shaw [DESHAW] (deep concepts, precise reasoning)
- ⭐ Explain virtual memory end to end: from a load instruction to data (TLB → page walk → fault → frame → restart) (09 §2.2 + COA 07)
- ⭐ Fork puzzles with `&&`/`||` and buffering (02 §3 N2–N4)
- Why does LRU not suffer Belady's anomaly? (stack property, 09 §2.4)
- Is an unsafe state a deadlock? (07 §2.5)
- Implement a semaphore using a mutex + condition variable (05 §4)
- Readers-writers: make it fair (06 §2.2)
- Why can't you call `printf` in a signal handler? (12 §2.7)
- What is the cost of a syscall and how does vDSO avoid it? (01 §2.5)
- Scheduling: prove SJF minimises average waiting time (exchange argument: swapping a longer job ahead of a shorter one never decreases total wait)
- Memory ordering: does Peterson's algorithm work on x86? (05 §2.3, COA 10)

## 4. HFT: IMC, Optiver, Tower, Graviton, Quadeye [HFT]
- ⭐ How do you make a thread's latency predictable on Linux? (13 §2.6: isolcpus, nohz_full, pinning, IRQ affinity, mlockall, huge pages, C-states)
- ⭐ What's the cost of: a syscall, a context switch, a minor and a major page fault, a cache miss, a mutex lock (uncontended/contended)? (17 cheat sheet)
- ⭐ Fastest IPC between two processes on one machine? Design it (12 §2.4: shm + SPSC ring)
- ⭐ Spinlock vs mutex for a 200 ns critical section? (05 N3)
- epoll vs busy polling vs kernel bypass (11 §2.5, §2.9)
- How do you avoid page faults during trading? (09 §2.7)
- Why is `malloc` avoided on the hot path? Alternatives? (08 §2.6)
- What is a futex? (05 §2.6)
- Seqlock: how does it work, and why are readers cheap? (05 §2.9)
- Your p99.9 latency has spikes: list the possible OS causes (13 §2.6)
- How do you measure 100 ns precisely? (13 §2.8)
- `SCHED_FIFO` dangers (04 §2.5)
- False sharing and its fix (COA 10 §2.6)
- Why bare metal rather than VMs/containers for trading? (15 §2.7)

## 5. NVIDIA [NVIDIA] (system software, drivers, CUDA runtime)
- ⭐ User vs kernel mode; how does a user program talk to a GPU driver? (ioctl, mmap, DMA, interrupts: 11 §2.3)
- ⭐ Pinned vs pageable memory; why is pinned faster for `cudaMemcpy`? (09 §2.7)
- Unified memory and GPU page faults (09 §2.7)
- What can't you do in kernel/interrupt context? (13 §2.2)
- Top half vs bottom half (11 §2.3)
- How does mmap work? (09 §2.6)
- Physically contiguous memory and DMA; IOMMU (08 §2.5)
- Virtual memory, TLB, huge pages (COA 07)
- Spinlocks in drivers; `spin_lock_irqsave` (05 §2.4, 13 §2.1)
- GPU sharing: MIG, vGPU, MPS (15 §2.4)
- Threads vs CUDA threads/warps (03 §2.9)
- Linux boot and driver probe; kernel modules (01 §2.8, 13 §2.1)

## 6. Qualcomm [QCOM] (embedded SW, modem, Android, drivers)
- ⭐ Priority inversion and its solutions (14 §2.5)
- ⭐ Mutex vs semaphore (especially in an RTOS / ISR context) (05 §2.7, 14 §2.3)
- ⭐ ISR design rules; deferred processing (14 §2.6, 11 §2.3)
- `volatile`: when is it needed? Is it enough for threads? (COA 10 §3.7)
- Process vs thread; context switch (02, 03)
- Virtual memory and paging (09)
- Android: Binder IPC, low-memory killer, zRAM (12 §2.6, 09 §2.8)
- Device driver structure: probe, file_operations, ioctl (11 §2.3)
- Spinlock vs mutex in the kernel; interrupt context (05, 13)
- Boot flow on an SoC: Boot ROM → SBL → … → Linux (01 §2.8)
- Write a thread-safe ring buffer / producer-consumer in C (06)
- Deadlock conditions; lock ordering (07)

## 7. Texas Instruments [TI] (embedded / firmware)
- ⭐ RTOS vs GPOS; hard vs soft real-time (14 §2.1–2.2)
- ⭐ RMS schedulability numerical; response-time analysis (14 §3)
- ⭐ What happens on reset of an MCU before `main()`? (01 §2.8, 08 §2.8: vector table, .data copy, .bss zero)
- Interrupt latency: what contributes? (14 §2.6)
- Shared variable between an ISR and main: `volatile` + atomicity (14 §4)
- Watchdog timer usage (14 §2.7)
- Polling vs interrupts vs DMA (11 §2.2)
- Static vs dynamic memory in embedded; stack overflow detection (08 §2.8, 14 §2.7)
- Semaphores/queues in FreeRTOS / TI-RTOS (14 §2.3)
- Disabling interrupts as a lock: when is it OK? (05 §2.4)

## 8. Apple [APPLE] (OS / platform / systems)
- ⭐ What is XNU? (Mach + BSD + IOKit, hybrid) (01 §2.6)
- ⭐ Virtual memory and memory pressure on iOS: compression, jetsam, no disk swap (09 §2.8)
- Grand Central Dispatch: queues vs threads; QoS classes; P/E cores (03 §2.7, 04 §2.5)
- APFS: copy-on-write, snapshots, clones (10 §2.6)
- Mach ports / XPC (12 §2.6)
- Rosetta 2 and TSO mode (COA 10 §3.4)
- Priority inversion (and how QoS/priority donation addresses it) (14 §2.5)
- Process lifecycle, fork/exec, signals (02, 12)
- Locks: `os_unfair_lock`, spin vs sleep (05)
- Durability of file writes (`fsync`, F_FULLFSYNC on macOS flushes the drive cache) (10 §2.7)

---

## 9. Rapid-fire one-liners (30-second answers)
| Q | A |
|---|---|
| Can a zombie be killed? | No: it's dead; reap via its parent (wait) or kill the parent |
| Who adopts orphans? | init (PID 1) or a subreaper |
| Signal that can't be caught? | SIGKILL, SIGSTOP |
| Default thread stack (Linux)? | 8 MB (main), configurable |
| Kernel stack size (x86-64)? | 16 KB |
| Page size? | 4 KB (huge: 2 MB, 1 GB; Apple M-series: 16 KB) |
| Minimum resources to avoid deadlock (n procs, max k)? | n(k − 1) + 1 |
| RMS bound for n = 2, 3, ∞? | 0.828, 0.780, 0.693 |
| EDF schedulable iff? | U ≤ 1 |
| Which algorithm has Belady's anomaly? | FIFO |
| Optimal page replacement? | OPT/MIN (farthest future use) |
| Optimal avg waiting time scheduling? | SJF / SRTF |
| select fd limit? | 1024 (FD_SETSIZE) |
| PIPE_BUF (atomic write)? | 4096 B on Linux |
| Default pipe capacity (Linux)? | 64 KB |
| `fork()` return in child? | 0 |
| What does `exec` keep? | PID, open fds (without CLOEXEC) |
| Is `printf` a syscall? | No (calls `write`) |
| Linux scheduler for normal tasks? | CFS → EEVDF (6.6+) |
| Futex fast path? | user-space atomic CAS, no syscall when uncontended |
