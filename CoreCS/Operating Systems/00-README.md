# Operating Systems — Index

> Asked in **every** company's interviews, but each one asks differently:
> - **Amazon / D. E. Shaw:** concept rounds: process vs thread, deadlock, scheduling, virtual memory, "what happens when…".
> - **HFT (IMC, Optiver, Tower, Graviton, Quadeye):** *cost* of OS operations: syscalls, context switches, page faults, CPU pinning, epoll vs busy-polling, shared-memory IPC, kernel bypass.
> - **NVIDIA / Qualcomm (system SW, drivers):** user vs kernel mode, interrupts, DMA, mmap, page-locked memory, Linux kernel basics, device drivers.
> - **TI / Qualcomm (embedded SW):** RTOS, ISRs, priority inversion, scheduling analysis (RMS/EDF), boot flow.
> - **Apple:** XNU (Mach + BSD), virtual memory and memory pressure, scheduling/QoS, file systems (APFS).
>
> **Depth:** software / system-software roles. Hardware mechanisms (TLB, caches, coherence, DMA hardware) are in the [COA notes](../Computer%20Organization%20and%20Architecture/00-README.md) and linked, not repeated.

## Study order
| # | Note | Core idea | Time | Must for |
|---|---|---|---|---|
| 01 | [OS Basics, Kernel & System Calls](01-OS-Basics-Kernel-and-System-Calls.md) | user/kernel mode, traps/interrupts, syscall path, kernel types, boot | 2 h | All, HFT, NVIDIA |
| 02 | [Processes](02-Processes.md) | PCB, states, context switch, fork/exec/wait, zombie/orphan | 3 h | **All** |
| 03 | [Threads](03-Threads.md) | thread vs process, kernel/user threads, pthreads, thread pools | 2 h | All |
| 04 | [CPU Scheduling](04-CPU-Scheduling.md) | FCFS/SJF/SRTF/RR/MLFQ, Gantt numericals; real schedulers from scratch: run queue vs wait queue, scheduling classes, CFS/EEVDF vruntime, when `schedule()` runs, multicore balancing, history (O(n)→O(1)→CFS→EEVDF), Windows/macOS/Android | 5 h | Amazon, DE Shaw, TI/QCOM |
| 05 | [Synchronization Primitives](05-Synchronization-Primitives.md) | race, Peterson, spinlock, mutex/futex, semaphore, condvar, seqlock | 4 h | **All**, HFT |
| 06 | [Classic Synchronization Problems](06-Classic-Synchronization-Problems.md) | producer-consumer, readers-writers, dining philosophers, coding problems | 3 h | Amazon, QCOM, DE Shaw |
| 07 | [Deadlocks](07-Deadlocks.md) | Coffman conditions, RAG, Banker's algorithm, lock ordering | 2 h | **All** |
| 08 | [Memory Management](08-Memory-Management.md) | allocation, fragmentation, paging/segmentation, buddy/slab, malloc | 3 h | All, HFT |
| 09 | [Virtual Memory: Page Faults & Replacement](09-Virtual-Memory-Page-Faults-and-Replacement.md) | demand paging, page cache vs anonymous pages, fault path, FIFO/LRU/OPT/Clock, thrashing, mmap, pinning (mlock vs DMA pin) | 3 h | **All** |
| 10 | [File Systems](10-File-Systems.md) | fd/open-file/inode tables, links, allocation, journaling, fsync | 3 h | Amazon, Apple, DE Shaw |
| 11 | [I/O Systems, I/O Models & Disk Scheduling](11-IO-Systems-IO-Models-and-Disk-Scheduling.md) | driver stack, blocking/non-blocking/epoll/io_uring, zero-copy, SCAN/C-SCAN | 3 h | HFT, NVIDIA, Amazon |
| 12 | [IPC & Signals](12-IPC-and-Signals.md) | pipes, shared memory, message queues, sockets, signals | 2 h | HFT, Amazon, QCOM |
| 13 | [Linux Internals & Low-Latency Tuning](13-Linux-Internals-and-Low-Latency-Tuning.md) | task_struct, IRQ top/bottom halves, perf tools, isolcpus, pinning | 3 h | **HFT**, NVIDIA, QCOM |
| 14 | [RTOS & Embedded OS](14-RTOS-and-Embedded-OS.md) | RMS/EDF, response-time analysis, priority inversion, ISR rules | 3 h | **TI, QCOM**, Apple |
| 15 | [Virtualization & Containers](15-Virtualization-and-Containers.md) | hypervisors, VT-x, EPT, namespaces, cgroups, Firecracker | 2 h | **Amazon**, NVIDIA |
| 16 | [Interview Question Bank](16-Interview-Question-Bank.md) | company-wise questions with pointers | ongoing | All |
| 17 | [Cheat Sheet](17-Cheat-Sheet.md) | formulas, numbers, one-liners | revision | All |

**Total ≈ 45 hours** for a thorough first pass (≈ 2–3 weeks at 3 h/day alongside DSA).

## Links to COA (read these alongside)
| OS topic | COA note to recall |
|---|---|
| System calls, interrupts, DMA | [COA 13 I/O, Interrupts & DMA](../Computer%20Organization%20and%20Architecture/13-IO-Interrupts-DMA.md) |
| Paging, TLB, page tables, CoW, huge pages | [COA 07 Virtual Memory & TLB](../Computer%20Organization%20and%20Architecture/07-Virtual-Memory-and-TLB.md) |
| Atomics, fences, memory model, false sharing | [COA 10 Coherence & Consistency](../Computer%20Organization%20and%20Architecture/10-Cache-Coherence-and-Memory-Consistency.md) |
| Low-latency systems | [COA 14 Low-Latency Hardware](../Computer%20Organization%20and%20Architecture/14-Low-Latency-Hardware-for-HFT.md) |

## How to study each note
1. Read *Intuition* + *Deep Dive*.
2. Solve the *Worked Numericals* yourself before reading the solution.
3. Answer the *Interview Questions* **out loud**.
4. Ask Claude: *"mock interview me on deadlocks, Amazon style"* or *"give me 5 scheduling numericals"*.
5. Update the progress table in the root [README](../../README.md).

## Recommended references
| Resource | Use for |
|---|---|
| Silberschatz, Galvin, Gagne — *Operating System Concepts* | Standard theory + numericals (scheduling, Banker's, page replacement) |
| Remzi & Andrea Arpaci-Dusseau — *OSTEP* (free online) | **Best intuition**: virtualization, concurrency, persistence |
| Robert Love — *Linux Kernel Development* | Linux internals (scheduler, interrupts, memory) |
| Michael Kerrisk — *The Linux Programming Interface* | syscalls, processes, signals, IPC, epoll (HFT/systems) |
| Brendan Gregg — *Systems Performance* | perf, latency analysis, tools |
| FreeRTOS docs / Jane Liu *Real-Time Systems* | RTOS, RMS/EDF |
| MIT 6.S081 (xv6) lectures | See a tiny real kernel end to end |
