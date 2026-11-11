# 04. CPU Scheduling

> **Asked at:** [AMZN] [DESHAW] compare algorithms, starvation, Gantt-chart numericals (also in OAs) · [TI] [QCOM] priority-based preemptive scheduling (RTOS, note 14) · [HFT] CPU affinity, `SCHED_FIFO`, why the scheduler is the enemy of latency · [APPLE] QoS classes, P-cores vs E-cores.
> **Prerequisites:** [02 Processes](02-Processes.md) (states, context switch).

---

## 1. Intuition
The scheduler is a **doctor's receptionist** with one doctor (one CPU core) and a waiting room (the ready queue). Different policies:
- **FCFS**: first come, first served. A patient with a 2-hour checkup makes everyone with a 2-minute question wait (**convoy effect**).
- **SJF**: shortest appointment first. Lowest average wait, but long cases may never get seen (**starvation**).
- **Round Robin**: everyone gets 10 minutes, then goes to the back of the line. Fair, and no one waits too long to be *seen*.
- **Priority**: emergencies first.

The scheduler decides **which ready thread runs next on each core, and for how long**.

---

## 2. Deep Dive

### 2.1 Metrics and formulas (memorise)
| Metric | Formula | Want |
|---|---|---|
| **Turnaround time (TAT)** | completion time − arrival time | low |
| **Waiting time (WT)** | TAT − burst time (time spent in the ready queue) | low |
| **Response time (RT)** | first time it gets the CPU − arrival time | low (interactive) |
| Throughput | processes completed per unit time | high |
| CPU utilisation | busy time / total time | high |
For non-preemptive algorithms, RT = WT. Process behaviour alternates between **CPU bursts** and **I/O bursts**. Interactive programs have short CPU bursts; number crunching has long ones.

### 2.2 Preemptive vs non-preemptive
- **Non-preemptive:** once running, a process keeps the CPU until it blocks or exits (FCFS, SJF).
- **Preemptive:** the OS can take the CPU away (timer interrupt, or a higher-priority arrival): SRTF, RR, preemptive priority, all modern OSes.
- **Dispatch latency:** time to stop one process and start another (the context switch, ~µs).

### 2.3 The algorithms
| Algorithm | Rule | Preemptive? | Pros | Cons |
|---|---|---|---|---|
| **FCFS** | order of arrival | no | simple, no starvation | **convoy effect**: short jobs stuck behind long ones ⇒ high avg WT |
| **SJF** | shortest next CPU burst first | no | **optimal average WT** (provable) for a given set | needs burst lengths (predict); long jobs can **starve** |
| **SRTF** | shortest **remaining** time; a new shorter arrival preempts | yes | optimal avg WT among preemptive | same prediction + starvation issues, more switches |
| **Priority** | highest priority first | either | expresses importance | **starvation** of low priority ⇒ fix with **aging** (raise priority while waiting) |
| **Round Robin (RR)** | each gets a time quantum q, then goes to the tail of the queue | yes | good response time; nobody waits more than (n−1)·q to run again | higher avg TAT; q choice matters |
| **Multilevel Queue** | separate queues (system / interactive / batch), each with its own algorithm, fixed priority between queues | yes | separates classes | starvation of low queues |
| **MLFQ** | processes **move between** queues based on behaviour | yes | approximates SJF **without knowing burst lengths** | tuning; gaming |

**Choosing the RR quantum:**
- q very large ⇒ RR becomes FCFS.
- q very small ⇒ too many context switches (overhead dominates).
- Rule of thumb: q larger than most CPU bursts (~80 % of bursts finish within one q); typical 1–100 ms.

**Predicting the next CPU burst for SJF (exponential averaging):**
τₙ₊₁ = α · tₙ + (1 − α) · τₙ, where tₙ = the actual last burst, τₙ = the last prediction, and 0 ≤ α ≤ 1 (α = 0.5 typical).

### 2.4 MLFQ rules (OSTEP version)
1. If priority(A) > priority(B), A runs.
2. Equal priority ⇒ round robin.
3. A new job starts at the **top** queue.
4. If a job **uses its whole time allotment** at a level, it moves **down** one queue (it looks CPU-bound).
5. Every period S, **boost** all jobs to the top queue (prevents starvation and handles jobs that become interactive).
Effect: short/interactive jobs finish quickly at high priority, while long CPU jobs sink to low priority with long quanta. That's SJF-like behaviour without knowing burst times. (Rule 4 counts *total* time at a level, so a job can't game it by yielding just before the quantum ends.)

### 2.5 What real OSes do (summary; full detail in §2.7–2.10)
**Linux CFS (Completely Fair Scheduler, 2.6.23–6.5):**
- Each task has **vruntime** = CPU time used, scaled by weight (nice value). A higher-weight (lower nice) task's vruntime grows more slowly.
- Always run the task with the **smallest vruntime** (the one that's had the least fair share). Tasks are kept in a **red-black tree** keyed by vruntime ⇒ O(log n) pick.
- Weights: nice 0 = 1024; each nice step ≈ ×1.25 (≈10 % CPU difference); nice −20 … +19.
- A sleeping task that wakes has a small vruntime ⇒ runs soon ⇒ good interactivity.

**Linux EEVDF (6.6+, 2023):** replaces CFS's pick rule with "Earliest Eligible Virtual Deadline First". It keeps fairness but gives latency-sensitive tasks better wake-up behaviour. Interview level: "CFS is fair-share by vruntime in an RB-tree; since 6.6 Linux uses EEVDF, a deadline-based refinement."

**Linux scheduling classes (priority order):**
| Class | Policy | Use |
|---|---|---|
| `SCHED_DEADLINE` | EDF with runtime/period/deadline | real-time with explicit deadlines |
| `SCHED_FIFO` | fixed priority 1–99, runs until it blocks/yields; no time slice | **RT / HFT** critical threads |
| `SCHED_RR` | like FIFO but round robin among equal priority | RT |
| `SCHED_OTHER` (normal) | CFS/EEVDF, nice values | everything else |
| `SCHED_IDLE` | only when nothing else wants to run | background |
⚠ A `SCHED_FIFO` thread in a busy loop can starve the whole core (Linux reserves 5 % by default via `sched_rt_runtime_us`).

**Multicore scheduling:**
- **Per-CPU run queues** (no single global lock) + periodic **load balancing** (migrating tasks between cores).
- **Processor affinity:** keep a task on the same core so its cache/TLB stay warm (soft affinity). **Hard affinity:** `taskset`, `sched_setaffinity`, `pthread_setaffinity_np` [HFT].
- **NUMA awareness:** prefer cores near the task's memory.
- [APPLE] **Asymmetric cores (P-cores / E-cores):** macOS/iOS use **QoS classes** (user-interactive, user-initiated, utility, background) to choose core type and priority. (Intel Thread Director does something similar on hybrid x86.)

### 2.6 Scheduling and latency [HFT]
- Every time the scheduler runs another task on your core, your thread loses its CPU (µs to ms) and returns to a cold cache.
- So HFT avoids the scheduler entirely: one thread per isolated core (`isolcpus`, `nohz_full`), pinned, busy-polling and never blocking. The scheduler then has nothing else to put on that core (note 13).

---

## PART B — HOW REAL SCHEDULERS ARE BUILT AND HOW THEY RUN

### 2.7 History: how real schedulers evolved (and why each was replaced)
| Era / OS | Scheduler | Data structure · cost to pick | Key idea | Why it was replaced |
|---|---|---|---|---|
| 1950s–60s batch (IBM OS/360) | FCFS / SJF / priority | job queue | maximise throughput | no interactivity |
| **1962 CTSS** (MIT, Corbató) | **first multilevel feedback queue** | several priority queues | demote CPU hogs, interactive jobs stay high | origin of MLFQ (part of Corbató's 1990 Turing Award) |
| 1970s–80s **Unix V7 / 4.3BSD** | priority with **decay of recent CPU usage** | 32 run queues (one bit per non-empty queue), RR with ~100 ms quantum | priority ≈ base + recent_cpu/4 + 2·nice; recent_cpu += 1 per tick, **decays** every second | recomputing every process each second is O(n); weak SMP support |
| **SVR4 / Solaris** | **table-driven** classes (TS, IA, SYS, RT) | a dispatch table per priority level: *quantum*, *new priority when the quantum expires*, *new priority after sleeping* | MLFQ made tunable by editing a table | still used in Solaris/illumos |
| **Windows NT → 11** | 32-level preemptive priority + **dynamic boosts** | ready queue per priority + bitmap ⇒ O(1) | boosts after I/O, foreground boost, anti-starvation boost | still used, refined (§2.9) |
| **Linux 2.4** (2001) | **O(n) "goodness"** scheduler with epochs | **one global run queue + one global lock** | each task has a time-slice counter; on every decision, **scan all runnable tasks** and compute `goodness()`; when all slices are used up, start a new epoch: `counter = counter/2 + priority` (sleepers keep half their unused slice = an interactivity bonus) | O(n) per decision; the global lock was a bottleneck on SMP |
| **Linux 2.6.0** (2003) | **O(1) scheduler** (Ingo Molnar) | **per-CPU** run queues, each with **two priority arrays** (active, expired) of 140 lists + a bitmap | pick = find first set bit (constant time); a task that used up its slice moves to the *expired* array; when *active* is empty, **swap the two pointers**; `sleep_avg` heuristic gives interactive tasks up to ±5 priority | interactivity heuristics were complex, gameable, and misclassified tasks ⇒ unfairness and stutters |
| FreeBSD 7+ (2003–08) | **ULE** | per-CPU queues | interactivity score from the sleep/run ratio | still FreeBSD's default |
| **Linux 2.6.23** (2007) | **CFS** (Molnar, inspired by Con Kolivas's RSDL) | per-CPU **red-black tree** keyed by vruntime | model an "ideal, perfectly fair CPU"; **no interactivity heuristics**: sleepers naturally have small vruntime | latency control still needed many knobs and wake-up heuristics |
| **macOS 11 / iOS 14** (2020) | **Clutch**, then **Edge** (Apple silicon) | hierarchy: QoS bucket → thread group (app) → threads | deadline-based choice between QoS levels, fairness between apps | current (§2.9) |
| **Linux 6.6** (2023) | **EEVDF** (Peter Zijlstra; Stoica & Abdel-Wahab 1995 paper) | augmented RB-tree (eligibility + virtual deadline) | fairness **plus** an explicit latency knob (a requested slice) | current |
| **Linux 6.12** (2024) | **sched_ext** | scheduling policy written as BPF programs | load custom schedulers at run time without rebuilding the kernel | current, optional |

**The O(1) scheduler's data structure** (still worth knowing: Linux's RT class and Windows use the same bitmap idea):
```
 per-CPU run queue
 ACTIVE array (tasks with slice left)          EXPIRED array (slice used up)
 bitmap: bit p = 1 if list p is non-empty      bitmap
 prio   0 → [ ]                                prio   0 → [ ]
   …                                             …
 prio 100 → [T3]   ← find-first-set-bit = 100 ⇒ run T3
 prio 120 → [T1, T5]                           prio 120 → [T2]   (T2 used its slice)
   …                                             …
 prio 139 → [ ]                                prio 139 → [ ]
 0–99 = real-time, 100–139 = normal (nice −20…+19)
 Active array empty?  swap(active, expired)  ⇒ O(1), no recalculation loop
```

### 2.8 How scheduling actually happens in modern Linux (step by step)

#### 2.8.1 The data structures
```
 ┌───────────────────────────── CPU 0 ─────────────────────────────┐   CPU 1, CPU 2 …
 │ struct rq  (per-CPU run queue, with its own lock)                 │   (each CPU has
 │   ├─ dl_rq  : RB-tree ordered by absolute deadline (EDF)          │    its own rq)
 │   ├─ rt_rq  : bitmap[100] + one FIFO list per RT priority         │
 │   ├─ cfs_rq : RB-tree of normal tasks by vruntime / virtual       │
 │   │           deadline (EEVDF), cached leftmost node              │
 │   ├─ curr   → the task running right now                          │
 │   └─ load/utilisation averages (PELT), clock, nr_running …        │
 └───────────────────────────────────────────────────────────────────┘
 pick_next_task(): ask each scheduling class in priority order
     stop  >  deadline  >  rt  >  fair (CFS/EEVDF)  >  ext (sched_ext)  >  idle
     → the first class that has a runnable task supplies the next task
```
- **Only runnable tasks are in a run queue.** A sleeping task (waiting on a socket, futex, timer or disk) sits on the **wait queue of the thing it waits for**, not in any run queue. That's why 10,000 sleeping threads cost the scheduler nothing.
- Each **scheduling class** is a table of function pointers (`enqueue_task`, `dequeue_task`, `pick_next_task`, `task_tick`, `check_preempt`…), like a C++ interface with one implementation per policy.
- `stop` is an internal class for migration/CPU hot-plug work; `idle` runs the idle loop when nothing else is runnable.

#### 2.8.2 When does the scheduler run? Only at these moments
| Trigger | Example | Voluntary? | What happens |
|---|---|---|---|
| **The task blocks** | `read()` on an empty socket, contended mutex (`futex_wait`), `sleep()`, waiting for a disk read | voluntary | the task sets its state to `TASK_INTERRUPTIBLE`/`UNINTERRUPTIBLE`, is **dequeued**, and calls `schedule()` **itself** |
| **The tick says its slice is used up** | a CPU hog ran its share | involuntary | the timer tick sets the **`TIF_NEED_RESCHED`** flag on the current task |
| **A more deserving task wakes up** | the editor wakes on a key press | involuntary | the wake-up code checks whether the woken task should preempt `curr`; if so, it sets `NEED_RESCHED` (sending a **reschedule IPI** if that CPU is a different one) |
| Yield / exit / priority change / migration | `sched_yield()`, `exit()`, `nice` | — | `schedule()` |

**Setting the flag is not switching.** The actual switch happens at the next **safe preemption point**:
- **return to user mode** from an interrupt or a syscall: the kernel checks the flag;
- inside the kernel (with `CONFIG_PREEMPT`): when `preempt_count` drops to 0 (no spinlock held) or on return from an interrupt into kernel code;
- explicit `cond_resched()` calls in long kernel loops (voluntary-preemption kernels).

#### 2.8.3 Walk-through: one CPU running a compiler and a text editor
```
t = 0      gcc (CPU-bound) is running. vim is asleep inside read() on the terminal:
           it's on the tty's wait queue, NOT in the run queue.
t = 1 ms   Timer tick → scheduler_tick → update_curr(gcc):
              runtime += 1 ms; vruntime += 1 ms × 1024/weight(gcc)
           Has gcc used its slice? No → return to gcc.
t = 2 ms   Same.
t = 3.2 ms Key pressed → keyboard IRQ → tty driver buffers the byte → wake_up(tty wait queue)
           → try_to_wake_up(vim):
               1. pick a CPU for vim (its previous CPU, whose cache is still warm, or an idle CPU nearby)
               2. enqueue vim in that CPU's cfs_rq; its vruntime is placed close to the queue's
                  minimum (it slept, so it hasn't used its share; it can't bank unlimited credit)
               3. check_preempt: vim is far "behind" gcc ⇒ set NEED_RESCHED on gcc
t = 3.2 ms IRQ handler returns → sees NEED_RESCHED → schedule():
               put_prev_task(gcc)     → gcc goes back into the RB-tree
               pick_next_task         → fair class → vim (earliest eligible deadline / leftmost)
               context_switch():
                 switch_mm()   → load vim's page-table base (CR3 / TTBR, with PCID/ASID: no full TLB flush)
                 switch_to()   → save gcc's registers + kernel stack pointer, load vim's
t = 3.25   vim runs ~50 µs, redraws, calls read() again → no more input → blocks:
               state = TASK_INTERRUPTIBLE, dequeued, schedule() → gcc continues
```
The editor got a ~0 ms response **without any "is this interactive?" heuristic**: it sleeps most of the time, so its vruntime stays low, and it wins whenever it wakes. That's the core insight of CFS.

#### 2.8.4 What the timer tick does on each CPU
1. `update_curr()`: charge the running task's runtime, advance its vruntime, update the run-queue's minimum vruntime.
2. Update **PELT** (per-entity load tracking): exponentially decaying averages of each task's load and utilisation (32 ms half-life). Used for load balancing and CPU frequency.
3. Check whether the current task's slice is over ⇒ set `NEED_RESCHED`.
4. RT: count down a `SCHED_RR` task's time slice.
5. Every so often, raise `SCHED_SOFTIRQ` to run **load balancing**.
With `nohz_full` and a single runnable task, the tick stops entirely (note 13).

#### 2.8.5 How CFS and EEVDF choose the task and size the slice
**CFS (2.6.23 – 6.5):**
- `vruntime += Δexec × 1024 / weight`. A heavier task's vruntime grows more slowly, so it gets more CPU.
- Pick = the **leftmost** node in the RB-tree (pointer cached ⇒ O(1) pick, O(log n) insert/remove).
- Slice = `sched_latency × weight / total_weight`.
  - `sched_latency` = 6 ms × (1 + log₂ CPUs, capped at ×4) = **24 ms** on 8+ CPUs.
  - Minimum granularity 0.75 ms × the same factor = **3 ms**. With too many tasks, the period stretches to n × 3 ms.
- A woken sleeper is placed at max(its old vruntime, `min_vruntime` − half the latency): a small bonus, but it can't hoard credit by sleeping for hours. A newly forked child starts slightly *behind*, so fork bombs can't starve everyone.

**EEVDF (6.6+):**
- Each task has a **lag** = the CPU it *should* have received minus what it actually got. It's **eligible** if lag ≥ 0 (it hasn't received more than its fair share).
- Each task has a **virtual deadline** = eligible time + (requested slice / weight).
- **Pick:** among eligible tasks, the one with the **earliest virtual deadline** (an RB-tree augmented with the minimum deadline per subtree ⇒ O(log n)).
- Default slice ≈ 3 ms. A latency-sensitive task can **request a shorter slice** (via `sched_setattr`, 6.12+) ⇒ an earlier deadline ⇒ it runs **sooner, in smaller pieces**, but its total share stays the same. This replaced many of CFS's wake-up heuristics.
- *Depth note:* the lag/eligibility arithmetic is beyond typical interviews. Know "**fair share (vruntime) + earliest virtual deadline; a short requested slice means lower latency, not more CPU**".

#### 2.8.6 Multicore: where a task runs and how load is balanced
**Placement on wake-up** (`select_task_rq_fair`): prefer the **CPU the task last ran on** (cache-hot), else an **idle CPU sharing the same last-level cache**. A "wake affine" heuristic may put it near the **waker** (producer/consumer pairs share data).

**Scheduling domains:** balancing is hierarchical because migration cost depends on distance.
```
 NUMA level   [ socket 0                       |  socket 1                 ]  balance rarely (memory is far)
 LLC level    [ core0  core1  core2  core3 ]     [ core4 … ]                  balance more often
 SMT level    [ cpu0 cpu1 ] (hyper-thread siblings)                           balance often, cheap
```
| Mechanism | When | What |
|---|---|---|
| Periodic balancing | from `SCHED_SOFTIRQ`, with intervals growing at higher domain levels | find the busiest group/CPU in the domain and **pull** tasks to even out load |
| New-idle balancing | a CPU is about to go idle | try to **pull** work first |
| NOHZ idle balancing | CPUs are idle with the tick stopped | one idle CPU balances on behalf of the others |
| Active balancing | a running task must move (e.g. it's on the wrong core type) | the per-CPU stopper thread **pushes** it |
| NUMA balancing | periodically | sample a task's memory accesses (by temporarily unmapping pages) and move the task or its pages so they're on the same node |
Tasks that ran very recently (≈ 0.5 ms, "cache-hot") are not migrated unless the imbalance is large.

#### 2.8.7 Group scheduling (cgroups) [AMZN]
Fairness is **hierarchical**: CPU is first split between **cgroups** by `cpu.weight`, then between the tasks inside each group. Each group appears as one schedulable entity in its parent's run queue.
- Example: user A runs 1 task and user B runs 9. Per-task fairness gives A 10 %; with one cgroup per user (equal weights), A gets **50 %**.
- **autogroup:** each terminal session becomes a group, so `make -j16` doesn't starve your desktop.
- **Kubernetes:** a CPU *request* becomes `cpu.weight` (relative share); a CPU *limit* becomes `cpu.max` (quota/period ⇒ throttling, note 15).

#### 2.8.8 The real-time and deadline classes
- **RT class (`SCHED_FIFO`/`SCHED_RR`):** a bitmap of 100 priorities + one list each ⇒ an **O(1)** pick (find first set bit). The O(1) design survives here because strict priority needs no fairness. On SMP, RT tasks are **pushed/pulled** between CPUs so that the N highest-priority RT tasks run on N CPUs. Default throttling leaves 5 % of each second for normal tasks.
- **Deadline class (`SCHED_DEADLINE`):** EDF on absolute deadlines (RB-tree) + **CBS** (constant bandwidth server): each task declares runtime/period and is limited to it. **Admission control** rejects new tasks if total bandwidth would exceed the limit, so deadlines are guaranteed.

#### 2.8.9 Frequency, energy and idle
- **schedutil** (cpufreq governor): sets each CPU's frequency from the scheduler's PELT utilisation. The scheduler and DVFS work together.
- **Energy Aware Scheduling (EAS, Linux 5.0+)** for big.LITTLE / hybrid CPUs: using an energy model of each core type, place a task where it costs the **least energy** while still meeting its needs. **uclamp** lets user space clamp a task's utilisation (Android boosts the foreground app's UI thread).
- **Idle:** if no class has a task, the idle task asks the **cpuidle governor** (menu/teo) which C-state to enter, and stops the tick (NO_HZ).

#### 2.8.10 What to know at interview depth vs beyond
| Know well (asked) | Know the name only (beyond typical depth) |
|---|---|
| per-CPU run queues; RB-tree by vruntime; scheduling classes in order | PELT decay formula, EEVDF lag maths |
| when `schedule()` runs: block, tick → NEED_RESCHED, wake-up preemption, preemption points | exact `sched_latency`/granularity tunables |
| why sleepers get low latency in CFS | domain balancing intervals, wake-affine details |
| wake-up path: choose CPU → enqueue → maybe preempt | sched_ext internals |
| load balancing idea: hierarchical domains, pull/push, cache-hot | Clutch/Edge internals |
| history: O(n) → O(1) → CFS → EEVDF, and the reason for each change | |

### 2.9 Other modern operating systems
**Windows (NT kernel) [AMZN]:**
- **32 priority levels:** 16–31 real-time (fixed), 1–15 variable (dynamic), 0 = the zero-page thread.
- Thread priority = the process's **priority class** (Idle 4, Below normal 6, **Normal 8**, Above normal 10, High 13, Realtime 24) + the thread's relative priority (−2 … +2, or idle / time-critical).
- **Dispatcher:** the highest-priority ready thread always runs and preempts lower ones immediately; round robin within a level. Ready queues per priority + a bitmap (the same idea as O(1)).
- **Quantum:** about 2 clock ticks on client Windows (~31 ms with a 15.6 ms tick), about 12 on servers. Foreground processes get **quantum stretching** on clients.
- **Dynamic boosts** (only in the 1–15 range), then decaying by **one level per quantum** back to the base:
  - after I/O completes: disk +1, network +2, keyboard/mouse +6, sound +8;
  - waking from an event or semaphore;
  - the foreground window's thread getting input.
- **Anti-starvation:** once a second, the "balance set manager" finds ready threads that have waited ~4 s and boosts them to 15 for one quantum. This also mitigates priority inversion.
- **Multiprocessor:** ideal processor + soft affinity, processor groups (64 CPUs each), Intel **Thread Director** hints for P/E cores (Windows 11).

**macOS / iOS (XNU) [APPLE]:**
- Mach threads have priorities 0–127. Normal apps default to ~31, kernel threads are higher, and real-time threads (audio) are at the top (~97+). Time-sharing priority **decays with CPU usage**, Mach-style.
- Apps express intent through **QoS classes**: user-interactive, user-initiated, default, utility, background, maintenance. QoS sets priority, **P-core vs E-core** placement, I/O priority and timer coalescing.
- **Clutch scheduler** (2020), a 3-level hierarchy per CPU cluster:
  ```
  root ──► QoS "root buckets"  (chosen EDF-style: higher QoS = earlier deadline, with a short "warp"
     │                           so UI work can jump ahead briefly, but low QoS can't starve)
     └──► thread groups (one per app/process group): fair share between apps, by interactivity score
            └──► threads inside the group: classic time-sharing
  ```
  ⇒ an app with 100 threads can't crowd out the UI of another app.
- **Edge scheduler** (Apple silicon): extends Clutch across P-clusters and E-clusters: which cluster a thread group runs on, and when to migrate or steal work between clusters.
- **Priority donation via turnstiles:** when a high-QoS thread waits on a lock held by a low-QoS thread, the holder temporarily gets the higher QoS (priority inheritance, note 14).

**FreeBSD ULE:** per-CPU run queues; an **interactivity score** from the ratio of sleep time to run time; interactive threads go to a high-priority "current" queue, batch threads to a time-sharing queue; work stealing between CPUs.

**Android [QCOM]:** the Linux scheduler (CFS/EEVDF + EAS) plus policy from user space:
- **cpusets** by app state: the top app may use the big cores; background apps are restricted to the little cores;
- **uclamp** boosts the foreground app's UI thread and RenderThread;
- Qualcomm's vendor kernels have long used **WALT** (Window-Assisted Load Tracking), which reacts to load changes faster than PELT, for frequency and placement decisions on Snapdragon big.LITTLE.

### 2.10 One-paragraph answer to "how does scheduling work in a modern OS?"
> Each CPU has its own run queue, with separate structures per scheduling class: deadline (EDF), real-time (a priority bitmap) and normal (an RB-tree ordered by weighted virtual runtime, using EEVDF's virtual deadlines in Linux 6.6+). Sleeping threads aren't in any run queue; they wait on the object they're blocked on. The scheduler runs when a thread blocks, when the timer tick finds that the running thread has used its fair slice, or when a woken thread deserves the CPU more. The latter two just set a need-resched flag, and the switch happens at the next safe point, such as return to user mode. Picking the next task asks the classes in priority order and takes the best candidate. The context switch then changes the page-table base and registers. Across CPUs, woken threads are placed on cache-warm or idle nearby cores, and periodic and idle-time balancing moves work through a hierarchy of SMT, cache and NUMA domains. cgroups make the fairness hierarchical. On top of that, frequency (schedutil) and energy-aware placement (EAS, or QoS on Apple platforms) decide *which kind* of core and *how fast* it runs.

---

## 3. Worked Numericals (Gantt charts)

**Process set used for N1–N4:**
| Process | Arrival | Burst |
|---|---|---|
| P1 | 0 | 8 |
| P2 | 1 | 4 |
| P3 | 2 | 9 |
| P4 | 3 | 5 |

**N1. FCFS**
```
| P1        | P2    | P3          | P4      |
0           8      12            21        26
```
| P | CT | TAT = CT − AT | WT = TAT − BT |
|---|---|---|---|
| P1 | 8 | 8 | 0 |
| P2 | 12 | 11 | 7 |
| P3 | 21 | 19 | 10 |
| P4 | 26 | 23 | 18 |
Avg TAT = 61/4 = **15.25**, Avg WT = 35/4 = **8.75**.

**N2. SJF (non-preemptive)**
At t=0 only P1 has arrived ⇒ P1 runs to 8. At t=8 the ready set is {P2:4, P3:9, P4:5} ⇒ P2, then P4, then P3.
```
| P1        | P2    | P4      | P3          |
0           8      12        17            26
```
| P | CT | TAT | WT |
|---|---|---|---|
| P1 | 8 | 8 | 0 |
| P2 | 12 | 11 | 7 |
| P3 | 26 | 24 | 15 |
| P4 | 17 | 14 | 9 |
Avg TAT = 57/4 = **14.25**, Avg WT = 31/4 = **7.75**.

**N3. SRTF (preemptive SJF)**
- t=0: P1 starts (remaining 8).
- t=1: P2 arrives with 4 < P1's remaining 7 ⇒ **preempt**, P2 runs.
- t=2, 3: P3 (9) and P4 (5) arrive; P2's remaining (3, then 2) is still the smallest ⇒ P2 continues to t=5.
- t=5: remaining: P1 7, P3 9, P4 5 ⇒ P4 runs to 10.
- t=10: P1 (7) runs to 17, then P3 to 26.
```
|P1| P2    | P4      | P1          | P3          |
0  1       5        10            17            26
```
| P | CT | TAT | WT | RT |
|---|---|---|---|---|
| P1 | 17 | 17 | 9 | 0 |
| P2 | 5 | 4 | 0 | 0 |
| P3 | 26 | 24 | 15 | 15 |
| P4 | 10 | 7 | 2 | 2 |
Avg TAT = 52/4 = **13**, Avg WT = 26/4 = **6.5**, Avg RT = 17/4 = 4.25.

**N4. Round Robin, q = 4** (convention: a newly arrived process joins the queue **before** the process just preempted)
- 0–4: P1 runs (remaining 4). By t=4, P2, P3, P4 have arrived ⇒ queue: P2, P3, P4, P1.
- 4–8: P2 runs, **done** (CT 8). Queue: P3, P4, P1.
- 8–12: P3 (remaining 5). Queue: P4, P1, P3.
- 12–16: P4 (remaining 1). Queue: P1, P3, P4.
- 16–20: P1, **done** (CT 20). Queue: P3, P4.
- 20–24: P3 (remaining 1). Queue: P4, P3.
- 24–25: P4 **done**. 25–26: P3 **done**.
```
| P1  | P2  | P3  | P4  | P1  | P3  |P4|P3|
0     4     8    12    16    20    24 25 26
```
| P | CT | TAT | WT | RT (first run − AT) |
|---|---|---|---|---|
| P1 | 20 | 20 | 12 | 0 |
| P2 | 8 | 7 | 3 | 3 |
| P3 | 26 | 24 | 15 | 6 |
| P4 | 25 | 22 | 17 | 9 |
Avg TAT = 73/4 = **18.25**, Avg WT = 47/4 = **11.75**, Avg RT = 18/4 = **4.5**.

**Comparison:**
| | FCFS | SJF | SRTF | RR (q=4) |
|---|---|---|---|---|
| Avg WT | 8.75 | 7.75 | **6.5** | 11.75 |
| Avg TAT | 15.25 | 14.25 | **13** | 18.25 |
| Max wait before first run | 18 (P4) | 15 | 15 | **9** |
SRTF wins on average waiting. RR has the worst average but **bounds** how long anyone waits to start, which is what interactive systems care about.

**N5. Exponential averaging.** α = 0.5, τ₀ = 10 ms, actual bursts 6, 4, 6 ms.
τ₁ = 0.5·6 + 0.5·10 = **8**; τ₂ = 0.5·4 + 0.5·8 = **6**; τ₃ = 0.5·6 + 0.5·6 = **6** ms.

**N6. RR overhead.** Context switch = 0.1 ms. Fraction of CPU wasted for q = 10 ms and q = 1 ms (assuming processes use the full quantum):
q = 10: 0.1 / (10 + 0.1) ≈ **1 %**. q = 1: 0.1 / 1.1 ≈ **9 %**.

**N7. CFS share.** Two CPU-bound tasks on one core: nice 0 (weight 1024) and nice 5 (weight 335).
Share = weight / Σweights: 1024/1359 ≈ **75 %** and 335/1359 ≈ **25 %**.

**N8. CFS slices and vruntime: why it's fair.** One CPU, 3 runnable tasks: A and B (nice 0, weight 1024), C (nice 5, weight 335). `sched_latency` = 24 ms.
- Total weight = 1024 + 1024 + 335 = 2383.
- Slices: A = B = 24 × 1024/2383 = **10.31 ms**; C = 24 × 335/2383 = **3.37 ms**.
- vruntime gained in one period: A = 10.31 × 1024/1024 = 10.31; C = 3.37 × 1024/335 = **10.30**.
⇒ Everyone's vruntime advances by the same amount. That *is* fairness in CFS: equal vruntime, unequal real CPU time according to weight.

**N9. Minimum granularity.** 10 runnable nice-0 tasks, `sched_latency` 24 ms, minimum granularity 3 ms.
24/10 = 2.4 ms < 3 ms ⇒ the period stretches to 10 × 3 = **30 ms**, each task gets 3 ms, and a task waits up to **27 ms** between turns. More runnable tasks ⇒ worse latency for each (another reason HFT keeps exactly one task per core).

**N10. Linux 2.4 epochs and the O(n) cost.** At the end of an epoch every task gets `counter = counter/2 + priority` (priority ≈ 20 ticks for nice 0).
- CPU hog (counter 0 left): 0/2 + 20 = **20**.
- Interactive task that slept with 12 left: 12/2 + 20 = **26** ⇒ more ticks and a higher goodness ⇒ it runs first after waking. Its counter can never exceed ~2 × priority (the series 20 + 10 + 5 + …).
- Cost: with 1000 runnable tasks at ~50 ns per `goodness()` evaluation, each scheduling decision takes **~50 µs** under one global lock ⇒ the reason for the O(1) rewrite.

**N11. Windows boost decay.** A Normal-class thread (base 8) wakes after keyboard input ⇒ +6 ⇒ **14**. If it keeps running, it drops one level per quantum: 13, 12, … back to **8 after 6 quanta**. A disk-I/O wake-up (+1) gives 9, back to 8 after one quantum.

**N12. Group scheduling.** One CPU; user A has 1 CPU-bound task, user B has 9.
- Per-task fairness: A = 1/10 = **10 %**.
- One cgroup per user, equal weights: A = **50 %**, each of B's tasks = 50 %/9 ≈ **5.6 %**.

**Try it:** non-preemptive **priority** (lower number = higher priority) on: P1 (AT 0, BT 10, pri 3), P2 (AT 1, BT 1, pri 1), P3 (AT 2, BT 2, pri 4), P4 (AT 3, BT 1, pri 5), P5 (AT 4, BT 5, pri 2). Find avg WT.
<details><summary>Answer</summary>P1 runs 0–10 (only one at t=0). At t=10: P2(1), P5(2), P3(4), P4(5) ⇒ P2 10–11, P5 11–16, P3 16–18, P4 18–19.
WT: P1 0, P2 10−1 = 9, P5 11−4 = 7, P3 16−2 = 14, P4 18−3 = 15 ⇒ sum 45 ⇒ <b>avg 9</b>.</details>

---

## 4. Interview Questions

### Basic
- **Q: Define turnaround, waiting and response time.** §2.1 formulas.
- **Q: Preemptive vs non-preemptive?** §2.2.
- **Q: What is the convoy effect?** In FCFS, short processes wait behind one long CPU-bound process ⇒ poor utilisation and high average wait.
- **Q: Which algorithm minimises average waiting time?** SJF (non-preemptive) / SRTF (preemptive). Follow-up: *"Why isn't it used directly?"* Burst lengths are unknown (predict with exponential averaging) and long jobs starve.

### Intermediate
- **Q: How does RR's quantum affect performance?** §2.3: too big ⇒ FCFS, too small ⇒ switching overhead.
- **Q: What is starvation and how is it fixed?** Low-priority/long jobs never run; fix with **aging**, or MLFQ priority boost.
- **Q: Explain MLFQ.** §2.4. It learns from behaviour: CPU hogs sink, interactive jobs stay high, a periodic boost prevents starvation.
- **Q: How does Linux CFS work?** vruntime, pick the minimum, red-black tree, nice weights. Since 6.6: EEVDF.

### Advanced / killer follow-ups
- **Q [HFT]: How would you make sure your trading thread is never descheduled?** Isolate a core (`isolcpus`/cpusets, `nohz_full`, `rcu_nocbs`), move IRQs away, pin the thread, optionally `SCHED_FIFO`, never block, avoid page faults (`mlockall`). Note 13.
- **Q: What's the danger of `SCHED_FIFO` priority 99 busy-polling?** It starves everything else on that core, including kernel threads (RCU, ksoftirqd) ⇒ possible system hangs. That's why the RT throttling default leaves 5 %, or you use an isolated core.
- **Q: Why per-core run queues instead of one global queue?** Lock contention on a global queue, plus cache affinity. The cost is load imbalance, fixed by periodic balancing and work stealing.
- **Q [APPLE]: How do iOS/macOS decide P-core vs E-core?** QoS class of the thread/queue (user-interactive → P-cores; background → E-cores), plus load.
- **Q: A thread is blocked in `read()` on a socket. Walk me through how it gets the CPU back when a packet arrives.** NIC interrupt → softirq processes the packet → data queued on the socket → wake up the socket's wait queue → `try_to_wake_up`: choose a CPU (cache-warm/idle nearby), enqueue in that CPU's run queue, check whether it should preempt the current task (set NEED_RESCHED, IPI if remote) → at the next safe point `schedule()` picks it → context switch → `read()` returns (§2.8.3).
- **Q: When exactly does Linux switch tasks?** On blocking (the task calls `schedule()` itself), or when NEED_RESCHED is set by the tick or a wake-up and the kernel reaches a preemption point: return to user mode, `preempt_count == 0` with `CONFIG_PREEMPT`, or `cond_resched()` (§2.8.2).
- **Q: Why did Linux go from O(n) → O(1) → CFS → EEVDF?** A global O(n) scan and lock didn't scale ⇒ per-CPU bitmap arrays (O(1)); those relied on fragile interactivity heuristics ⇒ CFS modelled ideal fairness with vruntime; CFS still needed latency heuristics ⇒ EEVDF added virtual deadlines and requestable slices (§2.7).
- **Q: Why is Linux's RT class still bitmap-based (O(1)) when normal tasks use an RB-tree?** Strict fixed priority needs only "highest non-empty level". Find-first-set-bit is perfect; fairness isn't needed.
- **Q: How does Linux decide which CPU a woken thread runs on?** Previous CPU if cache-hot, else an idle CPU in the same LLC domain, possibly near the waker; then periodic, idle and NUMA balancing correct imbalances (§2.8.6).
- **Q [AMZN]: How do Kubernetes CPU requests and limits map onto the Linux scheduler?** Requests → `cpu.weight` (proportional share in the cgroup hierarchy); limits → `cpu.max` quota/period (CFS bandwidth control ⇒ throttling).
- **Q [APPLE]: How does macOS keep the UI smooth while heavy background work runs?** QoS classes (UI = user-interactive, P-cores; background = E-cores, low priority), the Clutch scheduler's per-QoS deadlines and per-app fairness, and priority donation through turnstiles when the UI waits on a lock (§2.9).
- **Q [QCOM]: How does Android schedule on big.LITTLE Snapdragon chips?** EAS with an energy model, schedutil/WALT load tracking for frequency and placement, cpusets restricting background apps to little cores, and uclamp boosting the foreground UI and render threads.
- **Q: How does Windows keep interactive apps responsive?** Priority boosts after I/O (keyboard +6) that decay by one level per quantum, foreground quantum stretching, and an anti-starvation boost to 15 after ~4 s of waiting.
- **Q: Priority scheduling plus locks: what can go wrong?** Priority inversion (note 14: Mars Pathfinder, priority inheritance).

---

## 5. Common Traps
- Computing WT as start − arrival for preemptive algorithms. Use **TAT − burst**.
- Forgetting idle gaps when the first process arrives after 0, or when the CPU is idle between arrivals.
- RR queue order ambiguity: state your convention (new arrival before or after the preempted process).
- Claiming RR minimises average waiting time. It bounds *response* time, not average wait.
- Saying Linux uses "round robin" for normal tasks. It uses CFS/EEVDF; RR exists only as `SCHED_RR`.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Formulas | TAT = CT − AT · WT = TAT − BT · RT = first run − AT |
| FCFS | simple, convoy effect |
| SJF / SRTF | optimal avg WT; needs prediction τₙ₊₁ = αtₙ + (1−α)τₙ; starvation |
| Priority | starvation → aging |
| RR | quantum q: big ⇒ FCFS, small ⇒ overhead; bounded response |
| MLFQ | demote CPU hogs, periodic boost; SJF without knowing bursts |
| Linux | CFS (vruntime, RB-tree, nice weights ×1.25/step) → EEVDF (6.6+) |
| RT classes | `SCHED_DEADLINE` > `SCHED_FIFO`/`RR` (1–99) > `SCHED_OTHER` |
| Multicore | per-CPU queues, load balancing, affinity, NUMA |
| HFT | isolate + pin + busy-poll; no scheduler involvement |
| History | CTSS MLFQ (1962) → Unix decay priorities → Linux 2.4 O(n) epochs → 2.6.0 O(1) active/expired bitmaps → CFS 2.6.23 (2007) → EEVDF 6.6 (2023) → sched_ext 6.12 |
| Linux structures | per-CPU `rq`: dl (EDF tree), rt (bitmap[100]), cfs (RB-tree); classes stop > dl > rt > fair > ext > idle; sleepers on wait queues |
| When it switches | block ⇒ `schedule()`; tick / wake-up ⇒ NEED_RESCHED ⇒ switch at return to user / preemption point |
| Wake-up path | choose CPU (cache-hot / idle in LLC) → enqueue (vruntime near min) → maybe preempt (IPI) |
| Balancing | domains SMT → LLC → NUMA; periodic pull, new-idle pull, active push; avoid cache-hot migration |
| cgroups | hierarchical fairness by `cpu.weight`; `cpu.max` throttles |
| Windows | 32 levels, class + relative priority, boosts decay 1/quantum, anti-starvation boost to 15 |
| macOS/iOS | QoS classes; Clutch (QoS EDF → app fairness → threads), Edge for P/E clusters; turnstile donation |
| Android/QCOM | EAS + schedutil/WALT, cpusets, uclamp |
| Example set result | WT: FCFS 8.75, SJF 7.75, SRTF 6.5, RR(4) 11.75 |
