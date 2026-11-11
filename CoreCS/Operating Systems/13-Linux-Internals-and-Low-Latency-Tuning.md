# 13. Linux Internals & Low-Latency Tuning

> **Asked at:** [HFT] [DESHAW] "how do you make a thread's latency predictable on Linux?", jitter sources, isolcpus/nohz_full, perf · [NVIDIA] [QCOM] kernel vs user programming, interrupt context, kernel modules, preemption models · [AMZN] debugging a slow/hung server with Linux tools.
> **Prerequisites:** [01](01-OS-Basics-Kernel-and-System-Calls.md), [04 Scheduling](04-CPU-Scheduling.md), [09 Virtual Memory](09-Virtual-Memory-Page-Faults-and-Replacement.md), [11 I/O](11-IO-Systems-IO-Models-and-Disk-Scheduling.md), [COA 14 Low-Latency HW](../Computer%20Organization%20and%20Architecture/14-Low-Latency-Hardware-for-HFT.md).

---

## 1. Intuition
A general-purpose Linux box is a **busy shared office**: the scheduler moves people between desks, interrupts (phone calls) arrive at random desks, the cleaner (kernel housekeeping threads, timer ticks) walks by every millisecond, and the power-saving lights dim when it's quiet. That's great for **average throughput**, terrible for **predictable microsecond latency**.

Low-latency tuning gives your trading thread a **private soundproof room**: its own core, no phone calls (IRQs moved away), no cleaner (no tick), lights always on (no power saving), with everything it needs already on the desk (memory pre-faulted and locked).

---

## 2. Deep Dive

### 2.1 Kernel building blocks (names to know)
| Concept | One-liner |
|---|---|
| `task_struct` | the PCB; every process **and thread** is a task |
| Kernel threads | tasks that run only in kernel mode: `kworker` (workqueues), `ksoftirqd/N` (deferred softirqs), `kswapd` (page reclaim), `rcu_*`, `migration/N` |
| Kernel stack | small and fixed: **16 KB** on x86-64 (8 KB on older/32-bit) ⇒ no big local arrays or deep recursion in the kernel |
| Contexts | **process context** (syscall on behalf of a task: may sleep) vs **interrupt context** (hard IRQ / softirq: must **not** sleep) |
| Memory allocation | `kmalloc(size, GFP_KERNEL)` may sleep; `GFP_ATOMIC` never sleeps (interrupt context); `vmalloc` = virtually contiguous |
| Synchronization | spinlocks (`spin_lock_irqsave` when shared with an IRQ handler), mutexes (sleepable), RCU, atomics, per-CPU variables |
| Modules | `module_init`/`module_exit`, `insmod`/`rmmod`/`modprobe`, `lsmod`, `dmesg` for logs |

### 2.2 Kernel vs user-space programming [NVIDIA][QCOM]
| | User space | Kernel space |
|---|---|---|
| Library | libc, C++ STL | no libc (kernel's own `printk`, `kmalloc`, lists…) |
| Floating point / SIMD | free | not by default (`kernel_fpu_begin/end` needed) |
| Stack | 8 MB, grows | 16 KB fixed |
| Bug consequence | process crashes | **whole machine** crashes (oops/panic) |
| Memory | virtual, pageable, protected | kernel memory isn't paged; you can corrupt anything |
| Concurrency | threads you create | preemption, interrupts and all CPUs can enter your code at once |
| Sleeping | always allowed | only in process context, never with a spinlock held |

### 2.3 Preemption models
| Config | Meaning | Latency |
|---|---|---|
| `PREEMPT_NONE` | kernel code is never preempted (servers, throughput) | worst |
| `PREEMPT_VOLUNTARY` | preempt only at explicit points | medium (common distro default) |
| `PREEMPT` | kernel code is preemptible except in spinlocked/IRQ-off regions | better (desktops) |
| **`PREEMPT_RT`** | spinlocks become sleeping rt-mutexes with priority inheritance, IRQ handlers run in threads ⇒ almost everything is preemptible | **bounded, tens of µs worst case**; mainlined in Linux 6.12 (2024) |
HFT usually doesn't need PREEMPT_RT: they avoid the kernel on the hot path entirely. Industrial/robotics/automotive control (TI/QCOM embedded Linux) does use it.

### 2.4 Interrupt handling path
```
device raises IRQ → CPU enters the kernel → hard IRQ handler (top half: ack, minimal work)
    → raises a softirq (NET_RX, TIMER, BLOCK, TASKLET …)
    → softirqs run on IRQ exit (or in ksoftirqd/N under load)
    → wake the waiting task → scheduler → your thread runs
```
- `/proc/interrupts`: per-CPU IRQ counts. `/proc/softirqs`: per-CPU softirq counts.
- **IRQ affinity:** `/proc/irq/<n>/smp_affinity` (or `irqbalance`) chooses which cores handle which interrupts.

### 2.5 Timers and ticks
- `HZ` = tick frequency (100/250/1000). The tick does scheduler accounting, timers and RCU housekeeping.
- **NO_HZ_IDLE** (dyntick-idle): no ticks on idle CPUs (power saving).
- **`nohz_full`** (full dynticks): no tick on a CPU running **exactly one** task ⇒ removes the periodic interruption on the trading core.
- **hrtimers:** high-resolution timers (ns granularity) behind `nanosleep`, `timerfd`.

### 2.6 Jitter sources on a "quiet" core and their fixes [HFT]
| Jitter source | Typical hit | Fix |
|---|---|---|
| Other tasks scheduled on the core | µs–ms | **`isolcpus=`** / cpusets (`cset shield`), **pin** the thread (`taskset`, `pthread_setaffinity_np`) |
| Timer tick | ~1–5 µs every 1–4 ms | **`nohz_full=`** + only one runnable task on the core |
| RCU callbacks | µs | **`rcu_nocbs=`** (offload to other cores) |
| Device interrupts | µs | move IRQs off the isolated cores (`smp_affinity`), stop `irqbalance` from moving them back |
| Kernel threads / workqueues | µs | workqueue cpumask (`/sys/devices/virtual/workqueue/cpumask`), `kthread_cpus` |
| Page faults | 1 µs (minor) – 100 µs (major) | `mlockall`, pre-touch, huge pages, no `malloc` on hot path |
| TLB shootdowns (IPIs) from other threads' `munmap`/`mprotect` | µs | no memory-map changes during trading; isolate processes |
| CPU frequency changes | µs + slower code | `performance` governor, fixed frequency; consider disabling turbo for consistency |
| Deep C-states (sleep on idle) | 10–100 µs wake-up | busy-poll (never idle); `idle=poll` / `intel_idle.max_cstate=0`, `/dev/cpu_dma_latency` |
| SMT sibling sharing the core | variable | disable hyper-threading or keep the sibling idle |
| THP compaction / khugepaged | ms stalls | `transparent_hugepage=never` or `madvise` + explicit hugetlbfs |
| NUMA remote memory | +50–100 % latency | `numactl --cpunodebind --membind`, NIC on the same socket |
| SMIs (firmware interrupts) | 10s–100s µs, invisible to the OS | BIOS settings, choose good hardware; detect with `hwlatdetect` |
| Syscalls, locks, logging on hot path | µs | busy-poll, lock-free queues, async logging |

**Typical boot line for an HFT box** (example):
```
isolcpus=managed_irq,domain,2-7 nohz_full=2-7 rcu_nocbs=2-7 irqaffinity=0-1
intel_idle.max_cstate=0 processor.max_cstate=1 transparent_hugepage=never hugepages=1024
```
Then: pin each critical thread to one isolated core, set the `performance` governor, `mlockall`, pre-fault buffers, set `SCHED_FIFO` only if needed (on an isolated core it hardly matters), keep the NIC and memory on the same NUMA node, use kernel-bypass networking (note 11).

### 2.7 Observability: symptom → tool (asked at Amazon too)
| Question | Tool |
|---|---|
| What's using the CPU? | `top`/`htop` (press `1` for per-core), `mpstat -P ALL 1` |
| Where is the CPU time going inside my program? | **`perf top`**, `perf record -g` + `perf report`, flame graphs |
| IPC, cache misses, branch misses? | **`perf stat -e cycles,instructions,cache-misses,branch-misses`** |
| False sharing? | **`perf c2c`** |
| Which syscalls, how long? | `strace -c -f`, `strace -T`, `perf trace` |
| Library calls | `ltrace` |
| Memory usage / leaks | `free -m`, `/proc/meminfo`, `/proc/<pid>/status` (VmRSS), `smem`, `pmap`, Valgrind, ASan, heaptrack |
| Page faults | `perf stat -e page-faults,major-faults`, `/usr/bin/time -v` |
| Disk I/O | `iostat -x 1`, `iotop`, `blktrace` |
| Network | `ss -tanp`, `sar -n DEV`, `ethtool -S`, `tcpdump` |
| Context switches | `vmstat 1` (cs column), `pidstat -w`, `/proc/<pid>/status` (voluntary/nonvoluntary_ctxt_switches) |
| Kernel latency/tracing | `ftrace`, `trace-cmd`, **eBPF/bpftrace** (`runqlat`, `offcputime`, `biolatency`) |
| Scheduling latency | `perf sched latency`, `cyclictest` (RT latency benchmark) |
| Hung process | `cat /proc/<pid>/stack`, `gdb -p` → `thread apply all bt`, `wchan` in `ps` |
Brendan Gregg's **USE method**: for every resource check **U**tilisation, **S**aturation, **E**rrors.

### 2.8 Measuring latency correctly [HFT]
- Use `rdtsc`/`rdtscp` (invariant TSC) or `clock_gettime(CLOCK_MONOTONIC)` via vDSO. Serialise with `lfence; rdtsc` or `rdtscp` so the timestamp isn't reordered.
- Report **percentiles** (p50, p99, p99.9, max), not averages. Tail latency is what loses money.
- Use HdrHistogram-style recording. Beware coordinated omission (a stalled measurement loop hides the worst cases).
- Warm up first (caches, branch predictors, page faults, lazy binding: link with `-z now` / `LD_BIND_NOW=1`).

---

## 3. Worked Numericals

**N1. Tick jitter budget.** A trading loop must react in ≤ 3 µs at p99.9. HZ = 1000 with a 2 µs tick handler on the same core.
Ticks per second = 1000; one tick every 1 ms. A 3 µs reaction window overlaps a tick with probability ≈ (3 + 2) µs / 1000 µs ≈ 0.5 % ⇒ far above 0.1 % ⇒ the p99.9 is ruined by ticks alone ⇒ **`nohz_full`** is needed.

**N2. Page-fault cost on the hot path.** 5 µs order-handling path. A single minor fault (1 µs) adds 20 %. A major fault (100 µs) makes that order 21× slower ⇒ pre-fault + `mlockall`.

**N3. NUMA placement.** Local DRAM 80 ns, remote 140 ns. A path with 10 cache-missing loads: local 0.8 µs vs remote 1.4 µs ⇒ **+75 %** just from bad placement.

**N4. C-state wake-up.** Core idles in C6 (wake-up ~50 µs) between packets arriving every 100 µs on average ⇒ many packets pay +50 µs. Busy-polling keeps the core in C0 ⇒ 0 wake-up cost, at the price of 100 % CPU and power.

**Try it:** a strategy thread does 200,000 events/s. vmstat shows 2,000 non-voluntary context switches/s on its core at ~5 µs each (including cache refill). What fraction of events are hit, and what's the fix?
<details><summary>Answer</summary>Each switch disturbs ~1 event ⇒ 2,000/200,000 = <b>1 %</b> of events get +5 µs ⇒ it dominates p99. Fix: isolate the core (isolcpus/cpuset), pin the thread, move IRQs/kernel threads away, nohz_full.</details>

---

## 4. Interview Questions

### Basic
- **Q: What is a kernel module?** Code loaded into the running kernel (`insmod`), typically a driver; runs in kernel space with full privilege.
- **Q: What's different about writing kernel code?** §2.2 table.
- **Q: Which tools would you use to find why a program is slow?** `top` → `perf stat` → `perf record`/flame graph → `strace -c` → `iostat`/`vmstat` (§2.7).

### Intermediate
- **Q: Process context vs interrupt context?** Process context can sleep and access user memory (current task); interrupt context can't sleep and has no meaningful "current" process.
- **Q: `GFP_KERNEL` vs `GFP_ATOMIC`?** May sleep to reclaim memory vs never sleeps (interrupt context / spinlock held).
- **Q: What is `nohz_full`?** Stops the periodic tick on CPUs running a single task ⇒ less jitter.
- **Q: What does PREEMPT_RT change?** Spinlocks → rt-mutexes (sleepable, priority inheritance), threaded IRQs ⇒ bounded latency.

### Advanced / killer follow-ups
- **Q [HFT]: Your strategy has good median latency but terrible p99.9 spikes. What do you check?** §2.6 table: context switches (`pidstat -w`), IRQs on the core (`/proc/interrupts`), page faults, tick, C-states/frequency, THP compaction, TLB shootdowns, SMIs (`hwlatdetect`), logging/syscalls on the hot path, NUMA placement, GC/allocations.
- **Q [HFT]: Walk me through setting up a Linux box for low-latency trading.** BIOS (C-states off, HT decision, turbo/fixed frequency) → kernel boot params (isolcpus, nohz_full, rcu_nocbs, irqaffinity, hugepages) → IRQ affinity + no irqbalance on those cores → pin threads → mlockall + pre-fault → kernel-bypass NIC on the same NUMA node → measure with rdtscp percentiles.
- **Q [NVIDIA][QCOM]: Why can't you sleep while holding a spinlock?** Other CPUs spin waiting, possibly forever; and if an interrupt handler on the same CPU wants the lock ⇒ deadlock. Also preemption is disabled.
- **Q [AMZN]: A server is at 100 % CPU but throughput is low. How do you investigate?** `top` (user vs sys vs iowait vs softirq) → high sys ⇒ `strace -c`/`perf` for syscall hotspots; high softirq ⇒ network interrupts; user ⇒ `perf record` flame graph; also lock contention (`perf lock`, off-CPU analysis).
- **Q: How do you measure a 500 ns code path accurately?** `rdtscp`/`lfence;rdtsc`, warm-up, many iterations, percentiles, a pinned isolated core, subtract measurement overhead.

---

## 5. Common Traps
- Measuring averages instead of tail percentiles.
- Pinning a thread but leaving IRQs, kernel threads and the tick on that core.
- `SCHED_FIFO` busy loop on a non-isolated core ⇒ starving kernel threads.
- Forgetting THP compaction, TLB shootdowns and SMIs as jitter sources.
- Allocating memory or logging synchronously on the hot path.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Kernel basics | task_struct for all tasks; 16 KB kernel stack; process vs interrupt context |
| Kernel coding | no libc/FPU, can't sleep in IRQ or under spinlock, GFP_KERNEL vs GFP_ATOMIC |
| Preemption | none / voluntary / preempt / PREEMPT_RT (bounded latency, in mainline 6.12) |
| IRQ path | hard IRQ → softirq/ksoftirqd → wake task; affinity in /proc/irq |
| Ticks | HZ; NO_HZ idle; nohz_full for single-task cores |
| Jitter fixes | isolcpus, pin, nohz_full, rcu_nocbs, IRQ affinity, mlockall + pre-fault, huge pages, no THP, C-states off, performance governor, NUMA local |
| Tools | perf stat/record/c2c, strace, vmstat, pidstat, iostat, ftrace, bpftrace |
| Measurement | rdtscp, warm-up, percentiles (p99.9, max), coordinated omission |
