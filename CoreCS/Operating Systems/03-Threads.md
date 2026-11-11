# 03. Threads

> **Asked at:** [AMZN] [DESHAW] thread vs process, what's shared · [HFT] thread pinning, thread-per-core design, cost of creating threads · [NVIDIA] CPU threads vs GPU threads · [QCOM] [TI] user vs kernel threads, RTOS tasks · [APPLE] GCD / thread pools.
> **Prerequisites:** [02 Processes](02-Processes.md).

---

## 1. Intuition
A process is a **house**; threads are the **people living in it**. They share the kitchen, the fridge and the furniture (code, heap, globals, open files), but each person has their own notebook of what they're doing (registers, PC) and their own desk (stack). Adding a person to a house is much cheaper than building a new house, but because they share the fridge, they have to coordinate (synchronization, note 05).

---

## 2. Deep Dive

### 2.1 What threads share and what they don't
```
                     ONE PROCESS
┌───────────────────────────────────────────────────────┐
│ SHARED: code (.text), globals (.data/.bss), heap,      │
│         open files/sockets, signal handlers, PID,     │
│         current directory, page tables                │
│                                                       │
│  Thread 1          Thread 2          Thread 3         │
│ ┌──────────┐      ┌──────────┐      ┌──────────┐      │
│ │ PC, regs │      │ PC, regs │      │ PC, regs │ PRIVATE
│ │ stack    │      │ stack    │      │ stack    │      │
│ │ TLS,errno│      │ TLS,errno│      │ TLS,errno│      │
│ │ sig mask │      │ sig mask │      │ sig mask │      │
│ └──────────┘      └──────────┘      └──────────┘      │
└───────────────────────────────────────────────────────┘
```
- **Private per thread:** registers + PC, **stack**, thread-local storage (`thread_local`, `errno`), signal mask, scheduling priority, thread ID.
- **Shared:** everything else. A pointer to a local variable *can* be given to another thread (stacks are in the same address space), but it becomes dangling when that function returns.

> **Is the outer box the PCB and each thread box a TCB?** Conceptually yes: the PCB **tracks** the shared (process-wide) things and each TCB **tracks** one thread's private state. But PCB/TCB are **kernel bookkeeping records** in kernel memory that hold **pointers and saved values**. The code, heap and stacks themselves live in the process's **user address space**.
> ```
>  KERNEL MEMORY                                   USER ADDRESS SPACE
>  PCB: PID, parent, uid · page-table ptr ───────► code, globals, heap
>       fd table · signal handlers · cwd · limits
>       threads: TCB1, TCB2, TCB3
>  TCB1: TID, state, priority, saved PC+regs, saved SP ──► stack of T1
>        TLS base (errno), signal mask, kernel-stack ptr
>  TCB2: … saved SP ──► stack of T2         TCB3: … saved SP ──► stack of T3
> ```
> - The **stack is not inside the TCB**. The TCB holds the saved **stack pointer** (and saved registers while not running).
> - **"List of threads" = pointers to the TCBs** (a linked list: `PCB.thread_list → TCB1 ⇄ TCB2 ⇄ TCB3`); each TCB points back to its PCB. `pthread_create` allocates a TCB and inserts it; a thread exiting removes it; `exit()`/`kill` walks the list. Linux links every thread's `task_struct` into a per-process list (+ a `group_leader` pointer); Windows EPROCESS has a `ThreadListHead` of ETHREADs.
> - **Every process has at least one TCB:** only threads run, so the **main thread is created with the process** (`fork` → exactly 1 thread; `CreateProcess` → first thread, even with `CREATE_SUSPENDED`). Thread count 0 ⇒ the process is finished.
>   - Returning from `main` ⇒ `exit()` ⇒ **all threads killed**. `pthread_exit()` in `main` (written instead of `return`) ⇒ only the main thread ends; the process lives until the others finish. Code and details: §2.5.1.
>   - The **zombie** is the last TCB: the main thread's `task_struct` stays as the record until the parent `wait()`s (note 02 §2.7).
>   - Linux kernel threads are the reverse case: a `task_struct` with no user address space (`mm = NULL`).
> | PCB (shared) | TCB (per thread) |
> |---|---|
> | PID, parent, uid/gid; address space / page tables; fd table; signal **handlers** + process-pending signals; cwd, limits; CPU accounting; list of threads | TID; state; saved PC/SP/registers; signal **mask** + thread-pending signals; priority, policy, CPU affinity; TLS base; kernel stack |
> - **Thread switch** = save one TCB, restore another; the PCB stays, so **no page-table switch** (Linux skips `switch_mm` when `mm` is the same) ⇒ cheaper than a process switch.
> - **Linux:** no separate PCB. **Every thread has its own `task_struct`** (≈ TCB), and all threads point to **shared** `mm_struct` (address space), `files_struct` (fds), `fs_struct` (cwd), `signal_struct`/`sighand_struct`, with the same TGID. `clone()` flags pick what's shared vs copied (§2.4).
> - **Windows:** EPROCESS + PEB ≈ PCB; ETHREAD/KTHREAD + TEB ≈ TCB.
> - The open-file **offset** is in neither: it lives in the system-wide open-file description that the fd table points to (note 10 §2.1).

### 2.1.1 How stack space is divided between threads
Not divided from one big stack: **each thread gets its own separate stack region** in the process's virtual address space.
```
 kernel space        (+ each thread's own KERNEL stack, 16 KB on x86-64)
 MAIN thread stack ↓ created by the kernel at exec (argv, envp); grows on demand up to ulimit -s (8 MB)
 ····· gap ·····
 mmap region:  [ struct pthread + TLS | Thread 2 stack ↓ (fixed, 8 MB) | GUARD PAGE ]
               [ struct pthread + TLS | Thread 3 stack ↓               | GUARD PAGE ]
               shared libraries, mmap'd files …
 heap ↑ · .bss / .data / .text
```
**What `pthread_create` does for the stack:**
1. Size = `pthread_attr_setstacksize()` or the default (glibc: the `ulimit -s` value, 8 MB).
2. `mmap` a fresh region of size + one guard page (or reuse a cached stack from an exited thread).
3. `mprotect` the **lowest** page to `PROT_NONE` ⇒ guard page (stacks grow down, so overflow hits it).
4. Put glibc's `struct pthread` (the user-level TCB) + static TLS at the **top** of the block (the FS register points there).
5. `clone(…, child_stack = top of the usable region)` ⇒ the kernel sets the new thread's SP; its `task_struct` saves that SP when it isn't running.
| | Main thread stack | Other threads' stacks |
|---|---|---|
| Created by | kernel at `exec` | glibc `mmap` in `pthread_create` |
| Size | **grows automatically** to `ulimit -s` | **fixed** at creation (`pthread_attr_setstacksize`) |
| Overflow protection | guard gap | **guard page** (`pthread_attr_setguardsize`) ⇒ SIGSEGV kills the **whole process** |
| RAM | lazily, per touched page | same: 8 MB is only a **virtual reservation** |
- **Why thread stacks don't grow:** other mappings may sit right below an `mmap` region, so there's no room. (Go goroutines start at 2 KB and are **copied** to a bigger region when they fill.)
- Threads can access each other's stacks (one address space), but `&local` dangles after the function returns.
- **Defaults:** Linux 8 MB · Windows 1 MB reserved · macOS main 8 MB / secondary **512 KB** · Java ~1 MB (`-Xss`) · Go 2 KB growable · RTOS tasks: chosen at creation (256 B – few KB) · Linux kernel stack 16 KB.
- **Cortex-M [TI][QCOM]:** **MSP** (main stack) for ISRs/kernel, **PSP** (process stack) per RTOS task; a context switch reloads PSP.
- **Numbers:** 1000 threads × 8 MB = 8 GB virtual but only ~20 MB physical if each touches 20 KB. **32-bit:** 3 GB / 8 MB ≈ **384 threads max** ⇒ use smaller stacks. Recursion 50,000 deep × 200 B frames = 10 MB > 8 MB ⇒ guard page ⇒ SIGSEGV.
- A huge local array can jump over one guard page ("stack clash"); `-fstack-clash-protection` probes each page.

### 2.2 Thread vs process
| | Process | Thread |
|---|---|---|
| Address space | own (isolated) | shared with sibling threads |
| Creation cost | higher: `fork` copies page tables (~50–100+ µs typical) | lower: new stack + kernel task (~10–20 µs) |
| Context switch | switch page tables ⇒ TLB/cache cold | same address space ⇒ cheaper |
| Communication | IPC (pipes, shared memory, sockets) | just read/write shared memory (+ locks) |
| Crash isolation | one process crashing doesn't kill others | one thread's segfault kills **the whole process** |
| Use when | isolation/security matters (Chrome tabs, worker processes) | sharing data with low overhead (servers, compute) |

### 2.3 User-level vs kernel-level threads
| | Kernel threads (1:1) | User-level threads (N:1) | Hybrid (M:N) |
|---|---|---|---|
| Who schedules | the kernel | a user-space library | library maps M user threads onto N kernel threads |
| Blocking syscall | blocks only that thread | **blocks all threads** of the process | blocks only one kernel thread |
| Multicore parallelism | ✅ | ❌ (kernel sees one thread) | ✅ |
| Switch cost | kernel entry (~µs) | very cheap (~tens of ns, no kernel) | cheap |
| Examples | **Linux NPTL pthreads**, Windows threads, `std::thread` | early "green threads" | **Go goroutines**, Erlang, Java virtual threads (Loom) |
Modern C/C++ on Linux = **1:1**: every `std::thread` is a real kernel task. **Coroutines** (C++20 `co_await`, async runtimes) are user-level tasks multiplexed onto a few kernel threads.

### 2.4 Linux view: threads are just tasks sharing things
Linux has one call, **`clone()`**, which takes flags saying what to share:
- `fork()` ≈ `clone()` sharing nothing (copy-on-write copy).
- `pthread_create()` ≈ `clone(CLONE_VM | CLONE_FILES | CLONE_FS | CLONE_SIGHAND | CLONE_THREAD …)`, sharing memory, files and signal handlers.

Every thread has its own `task_struct` and **TID**. `getpid()` returns the **TGID** (thread-group ID = the PID of the main thread), which is why all threads of a process show the same "PID". See them with `ps -L` or `/proc/<pid>/task/`.

### 2.5 pthreads / std::thread essentials
```cpp
#include <thread>
#include <vector>
void work(int id, std::vector<int>& out) { out[id] = id * id; }

int main() {
    std::vector<int> out(4);
    std::vector<std::thread> ts;
    for (int i = 0; i < 4; i++)
        ts.emplace_back(work, i, std::ref(out));   // std::ref: pass by reference
    for (auto& t : ts) t.join();                   // wait for all
}
```
| Operation | Meaning |
|---|---|
| `pthread_create` / `std::thread` | start a new thread running a function |
| `join` | wait for it to finish (and free its resources) |
| `detach` | let it run independently; resources are freed automatically when it ends |
| neither, before `std::thread` is destroyed | **`std::terminate()`**: program crash |
Classic bug: passing `&i` (the loop variable) to threads, so all threads see whatever `i` is when they read it. Pass by value.

#### 2.5.1 Ending threads: `return` from main vs `pthread_exit()` vs `exit()`
`pthread_exit()` is written **instead of** `return`, not before it. It never returns, so code after it is dead.
```c
void *worker(void *a) { sleep(2); printf("worker done\n"); return NULL; }

int main() {                              // A: return       B: pthread_exit       C: join
    pthread_t t; pthread_create(&t, NULL, worker, NULL);
    printf("main done\n");
    return 0;          // A → output: "main done" only (worker killed mid-sleep)
 // pthread_exit(NULL);// B → "main done", 2 s later "worker done"; process exits (status 0) after the last thread
 // pthread_join(t, NULL); return 0;   // C (recommended) → both lines; main waits explicitly
}
```
**Why `return` kills everything:** glibc's startup code does `exit(main(argc, argv));`, so returning from main ⇒ `exit()` ⇒ atexit handlers, C++ static destructors, stdio flush ⇒ the **`exit_group`** syscall ⇒ the kernel kills **all** threads. `pthread_exit()` never goes back to that code: it ends only the calling thread (plain `exit` syscall). When the last thread ends, the process exits as if `exit(0)` were called.
| You write | Threads that end | Process |
|---|---|---|
| `return n;` in main | **all** | ends, status n |
| `exit(n)` in **any** thread | **all** | ends, status n (cleanup + flush) |
| `_exit(n)` anywhere | **all**, no cleanup/flush | ends, status n |
| `pthread_exit()` in main | main only | continues; status 0 when the last thread ends |
| `return` / `pthread_exit()` in a worker | that worker only | continues |
- Prefer **join**. C++ `std::thread` has no `pthread_exit` equivalent: join or detach, and returning from main still ends everything.
- With `pthread_exit` in main, don't let workers use pointers to main's local variables (undefined behaviour once main's thread ends).

### 2.6 Thread safety, reentrancy, thread-local storage
- **Thread-safe:** correct when called from several threads at once (uses locks, atomics, or no shared state).
- **Reentrant:** can be safely re-entered (e.g. from a signal handler interrupting itself): uses no static/global state and no locks. Reentrant is **stronger** for signal handlers; a thread-safe function holding a lock is *not* reentrant (it would deadlock re-locking).
- Example: `strtok` keeps static state ⇒ not thread-safe. `strtok_r` takes the state as an argument ⇒ thread-safe and reentrant.

> **What "re-entered" means:** the function is called **again while an earlier call hasn't finished, on the SAME thread**. Usually a **signal handler** (or an ISR in embedded code) interrupts the first call halfway and calls the function again; the first call is frozen underneath until the handler returns.
>
> **Why a lock gives thread-safety but not reentrancy:**
> ```
> TWO THREADS:  A: printf → takes the stdout lock … B: printf → waits … A finishes, releases → B runs ✓
>               (the lock holder A keeps running, so it eventually releases)
> RE-ENTRY:     main: printf → takes the lock … ⚡ signal → handler (same thread): printf → waits for the lock
>               holder = the frozen printf underneath, which can't run until the handler returns ⇒ DEADLOCK ✗
> ```
> A recursive mutex doesn't help: the handler would then work on the half-updated buffer ⇒ corruption.
> ```c
> char *to_upper(const char *s) { static char buf[100]; /*fill*/ return buf; }      // neither
> void to_upper_locked(const char *s, char *out) { lock; /*use static buf*/ unlock; } // thread-safe only
> void to_upper_r(const char *s, char *out) { /*locals + caller's buffer only*/ }    // both
> ```
> | Function | Thread-safe | Reentrant | Why |
> |---|---|---|---|
> | `strlen`, `memcpy`, `to_upper_r` | ✅ | ✅ | parameters + locals only |
> | `strtok`, `localtime`, static-buffer functions | ❌ | ❌ | hidden shared static state |
> | `strtok_r`, `localtime_r` | ✅ | ✅ | the caller keeps the state |
> | `printf`, `malloc`, anything with a mutex | ✅ | ❌ | lock + shared state ⇒ deadlock/corruption on re-entry |
> | `thread_local` static buffer | ✅ | ❌ | the handler on the same thread uses the same copy |
> | lock-free `std::atomic` increment | ✅ | ✅ | one indivisible instruction, no lock |
> **"Reentrant is stronger":** its rules (no shared state **and** no locks) are stricter than thread-safety's (shared state OK if locked). Hence `printf`/`malloc` are banned in signal handlers; use only **async-signal-safe** functions (`write`, `_exit`, `sem_post`…; note 12 §2.7). [TI][QCOM] ISRs must call only reentrant code (beware static-buffer `sprintf`, a C library without per-task reent state).
- **`thread_local`** (C++11) / `__thread`: each thread gets its own copy. `errno` is thread-local, so threads don't overwrite each other's error codes. Great for per-thread counters/allocators with no false sharing.

### 2.7 How many threads? Thread pools
Creating a thread per request is wasteful (creation cost + too many threads ⇒ context switching + memory for stacks). A **thread pool** keeps N threads waiting on a task queue.
- **CPU-bound work:** N ≈ number of cores (more just adds switching).
- **I/O-bound work:** N ≈ cores × (1 + wait time / compute time).
- Each thread's stack reserves virtual memory (8 MB default on Linux, but physical pages are allocated lazily).
- [APPLE] **Grand Central Dispatch** (libdispatch): you submit blocks to queues and the OS manages a pool sized to the cores. [HFT] **thread-per-core**: one pinned thread per isolated core, each busy-polling its own work, with no pool and no sharing.

### 2.8 Threads + fork + signals (the gotchas)
- **fork in a multithreaded process:** the child gets **only the calling thread**. Mutexes held by other threads at fork time remain **locked forever** in the child (`malloc`'s lock, for example) ⇒ the child should only call async-signal-safe functions and then `exec`. (`pthread_atfork` exists but is hard to use correctly.)
- **Signals:** a process-directed signal (e.g. `SIGTERM`) goes to **any one thread** that doesn't block it. Common pattern: block signals in all threads and have one thread call `sigwait()` / use `signalfd` (note 12).

### 2.9 CPU threads vs GPU threads [NVIDIA]
| | CPU thread | GPU (CUDA) thread |
|---|---|---|
| Count | tens | tens of thousands in flight |
| Weight | heavy (OS task, kernel stack, µs switch) | lightweight: hardware-scheduled in **warps of 32**, switch is free (registers stay resident) |
| Latency hiding | caches, OoO | switch to another ready warp |
See [COA 12 GPU](../Computer%20Organization%20and%20Architecture/12-GPU-Architecture.md).

---

## 3. Worked Numericals

**N1. Thread-pool size for an I/O-bound service.** 8 cores; each request does 10 ms of CPU work and waits 40 ms on a database.
N = cores × (1 + W/C) = 8 × (1 + 40/10) = 8 × 5 = **40 threads**.
Check: each thread is busy 10/50 = 20 % of the time ⇒ 40 × 0.2 = 8 cores busy ✓.

**N2. Amdahl with threads.** 90 % of a job parallelises perfectly. Speed-up on 8 threads, and the maximum?
S(8) = 1 / (0.1 + 0.9/8) = 1 / (0.1 + 0.1125) = 1 / 0.2125 ≈ **4.7×**. Max (∞ threads) = 1/0.1 = **10×**.

**N3. Creation cost: per-request threads vs a pool.** 50,000 requests/s; creating + destroying a thread costs 20 µs.
Overhead = 50,000 × 20 µs = 1 s per second = **one whole core** just creating threads. A pool removes this.

**N4. Stack memory.** 2,000 threads with 8 MB default stacks.
Virtual reservation = 2000 × 8 MB = **16 GB** of address space (fine on 64-bit, since physical memory is used only for touched pages; on 32-bit it's impossible: max ~3 GB user space). Typical fix: smaller stacks (`pthread_attr_setstacksize`) or fewer threads.

**Try it:** 16 cores, tasks are 5 ms CPU + 15 ms waiting on network. Pool size? And if the work becomes pure CPU?
<details><summary>Answer</summary>16 × (1 + 15/5) = 16 × 4 = <b>64</b>. Pure CPU: ≈ <b>16</b> (one per core).</details>

---

## 4. Interview Questions

### Basic
- **Q: Process vs thread?** Table §2.2. Threads share an address space and resources; each has its own stack, registers and TLS.
- **Q: PCB vs TCB?** PCB = per-process record (address space, fds, handlers, PID, thread list); TCB = per-thread record (TID, state, saved registers/SP, mask, priority, TLS). Follow-up: *"Is the stack in the TCB?"* No: it's in user memory; the TCB holds the saved SP. *"Linux?"* Each thread is a `task_struct`; process parts are shared `mm`/`files`/`signal` structs (§2.1 box).
- **Q: What is shared between threads?** Code, globals, heap, open files, signal handlers. Not shared: stack, registers/PC, TLS, signal mask.
- **Q: Why use threads instead of processes?** Cheaper creation and switching, and easy data sharing. The cost is that sharing needs synchronization and one crash kills all threads.

### Intermediate
- **Q: User-level vs kernel-level threads?** §2.3. A blocking syscall blocks all user threads in N:1; there's no multicore parallelism; it's cheap. Linux pthreads are 1:1.
- **Q: How are threads implemented in Linux?** As tasks created with `clone()` and sharing flags; each has a TID; `getpid()` returns the TGID.
- **Q: Thread-safe vs reentrant?** Thread-safe = correct when called concurrently by different threads (locks OK). Reentrant = correct when re-entered on the same thread before the first call finishes (signal handler / ISR): no shared state and no locks. `strtok` (neither) vs `strtok_r` (both); `printf`/`malloc` are thread-safe but not reentrant (§2.6 box). Follow-up: *"Can you call `malloc` in a signal handler?"* No: deadlock on the heap lock, or corruption.
- **Q: How do you size a thread pool?** CPU-bound ≈ cores; I/O-bound ≈ cores × (1 + W/C).

### Advanced / killer follow-ups
- **Q: What happens if `main` returns while other threads are still running?** `return` from main → `exit()` → the whole process ends and all threads are killed mid-work. `join` them first, or use `pthread_exit()` in main to let them finish.
- **Q: What happens if one thread segfaults?** SIGSEGV kills the entire process, all threads included (unless a handler handles it, which is rarely safe).
- **Q: A multithreaded program forks. What's dangerous?** §2.8: only one thread is copied, held locks stay locked, so `malloc`/`printf` in the child can deadlock. Only exec.
- **Q: Why is a thread context switch cheaper than a process switch?** No page-table switch ⇒ no TLB flush; caches still hold the shared data.
- **Q [HFT]: Thread pool or thread-per-core for a trading engine?** Thread-per-core: pinned, isolated, busy-polling, no shared queues, no context switches ⇒ predictable µs latency. Pools suit throughput workloads, not tail latency.
- **Q: How is stack space allocated for threads? What happens on overflow?** The main stack is set up by the kernel and grows to `ulimit -s`; every other thread gets its own fixed-size `mmap`'d stack (default 8 MB, virtual only) with a guard page; overflow hits the guard page ⇒ SIGSEGV kills the whole process (§2.1.1). Follow-up: *"Why can't you create 1000 threads on 32-bit?"* 3 GB / 8 MB ≈ 384 ⇒ shrink the stacks.
- **Q: Does a thread's stack belong only to it?** Logically yes, but it's in the shared address space, so other threads *can* access it through pointers (dangerous after return).

---

## 5. Common Traps
- Saying threads share the stack.
- Forgetting `join`/`detach` on `std::thread` (⇒ `std::terminate`).
- Capturing the loop variable by reference when starting threads.
- Thinking more threads = faster for CPU-bound work beyond the core count.
- Assuming one thread's crash affects only that thread.
- Calling non-async-signal-safe functions in the child after fork in a multithreaded program.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Thread | execution path within a process; private regs/PC/stack/TLS |
| Shared | code, globals, heap, fds, signal handlers, page tables |
| Thread stacks | separate regions: main = kernel-made, grows to 8 MB; others = mmap'd, fixed (default 8 MB virtual), guard page ⇒ SIGSEGV; lazy RAM; macOS secondary 512 KB, Windows 1 MB |
| PCB / TCB | kernel records: PCB = process-wide (mm, fds, handlers); TCB = per thread (saved regs/SP, state, mask, TLS); stack lives in user memory; Linux: task_struct per thread + shared mm/files/signal |
| vs process | cheaper create/switch, no isolation, needs sync |
| Models | 1:1 (Linux pthreads), N:1 (green), M:N (goroutines) |
| Linux | `clone()` flags; TID per thread; `getpid()` = TGID |
| std::thread | must join or detach |
| Ending | return from main / `exit()` anywhere ⇒ `exit_group` ⇒ all threads die; `pthread_exit()` (instead of return) ⇒ only that thread; last thread ends ⇒ exit(0) |
| Thread-safe vs reentrant | thread-safe = concurrent threads OK (locks allowed); reentrant = re-entry on the same thread (signal/ISR) OK: no shared state, no locks; lock holder frozen underneath ⇒ deadlock; printf/malloc thread-safe, not reentrant; `strtok_r` both |
| Pool size | CPU: ≈ cores · I/O: cores × (1 + W/C) |
| fork + threads | only the caller is copied ⇒ exec immediately |
| HFT | thread-per-core, pinned, busy-poll |
