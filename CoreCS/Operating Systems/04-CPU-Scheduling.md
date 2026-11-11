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

### 2.5 What real OSes do (quick summary; Part B §2.7–2.17 explains every term from scratch)
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

## PART B — HOW A REAL OS SCHEDULER WORKS (Linux first, then others)

> **How to read Part B.** Each section uses only words defined in the sections before it, so read in order.
> - §2.7–2.11 are the core. They are what Amazon, D. E. Shaw, HFT and NVIDIA systems rounds ask about.
> - §2.12–2.15 are short, company-specific add-ons.
> - §2.16 separates the scheduler algorithm, its tunables and per-thread policy, and covers whether updates can change it.
> - §2.17 says what you can safely skip.
>
> The older, denser version is kept in [archive/04-CPU-Scheduling-original-full-PartB.md](archive/04-CPU-Scheduling-original-full-PartB.md).

### 2.7 The foundation: what the scheduler keeps track of

#### 2.7.1 "Task" = thread
Linux schedules **threads**, not processes. Each thread has its own `task_struct` (note 03 §2.1), and the kernel calls it a **task**. A single-threaded process is one task. In Part B, "task" and "thread" mean the same thing.

#### 2.7.2 The three states that matter to the scheduler
```
                     scheduler picks it
     RUNNABLE  ─────────────────────────────►  RUNNING   (on a CPU right now)
  (ready, waiting  ◄─────────────────────────
   for a CPU)        preempted: slice used up, or a more deserving task woke up
       ▲                                          │
       │ the awaited event happens                │ needs something not available yet
       │ (data arrives, lock released,            │ (read() with no data, mutex already
       │  timer expires)  = "WAKE-UP"             │  locked, sleep(5))  = "BLOCKS"
       │                                          ▼
       └───────────────────────────────────  SLEEPING (blocked)
```
These are note 02's READY / RUNNING / WAITING. Linux quirk: both RUNNING and RUNNABLE use the state value `TASK_RUNNING`. Sleeping is `TASK_INTERRUPTIBLE` (a signal can wake it) or `TASK_UNINTERRUPTIBLE` (it can't be woken, e.g. waiting for a disk read).

#### 2.7.3 Run queue = the list of RUNNABLE tasks of one CPU
The **run queue** is the real version of the textbook "ready queue". It holds every task that **could run right now but is waiting for the CPU**, plus a pointer to the task that **is** running.

- **One per CPU:** every core has its own run queue (`struct rq`). A 16-core machine has 16 run queues.
- **What it contains** (simplified):

| Field | What it is | Why it's needed |
|---|---|---|
| `lock` | a spinlock protecting this run queue | this CPU and other CPUs (waking a task here, stealing work from here) may change it at the same time |
| `curr` | pointer to the task running on this CPU now | to charge it CPU time and decide whether to preempt it |
| `nr_running` | how many runnable tasks are here | load balancing (§2.11) needs to know which CPU is busiest |
| the **containers** of runnable tasks | one per scheduling class (§2.8): an array of lists for real-time tasks, a red-black tree for normal tasks | so that "who runs next?" is fast |
| clock, recent load | this CPU's time and how busy it has been lately | to charge time and to balance |

- **What is NOT in it: sleeping tasks.** A task blocked on a socket is in no run queue; it is on that socket's **wait queue** (next subsection). So 10,000 sleeping threads cost the scheduler nothing, because it only looks at runnable ones.

#### 2.7.4 Wait queue = the list of tasks sleeping on one thing
Every object a task can wait for has its own **wait queue**, which is just a linked list of sleeping tasks. Examples: a socket, a pipe, a futex (the kernel side of a mutex), a disk request, a timer.
- **Block:** the task sets its state to sleeping → leaves the run queue → adds itself to the object's wait queue → calls the scheduler to give the CPU away.
- **Wake-up:** when the event happens (e.g. a packet arrives for that socket), the kernel code handling it walks that wait queue and puts each waiting task back into **a** run queue (§2.10.4).

```
 CPU 0 run queue                           CPU 1 run queue
 ┌───────────────────────────────┐         ┌───────────────────────────────┐
 │ curr     → gcc                │         │ curr     → firefox            │
 │ runnable → make, python       │         │ runnable → (none)             │
 └───────────────────────────────┘         └───────────────────────────────┘

 Wait queues (they belong to the objects, not to any CPU)
   socket 7   : [ nginx-worker ]          ← sleeping in read() / epoll_wait()
   tty1       : [ vim ]                   ← waiting for a key press
   futex 0x…  : [ worker-3, worker-4 ]    ← waiting for a mutex
   disk req   : [ updatedb ]              ← waiting for a block to be read
```

#### 2.7.5 Why one run queue per CPU instead of one global queue?
| One global queue (Linux 2.4) | One queue per CPU (Linux 2.6+, Windows, macOS) |
|---|---|
| every CPU takes **the same lock** on every scheduling decision, so on 64 cores they queue up waiting for it | each CPU mostly touches only its own queue and lock, so it scales |
| a task runs on whichever CPU grabs it next, so its data is usually in another core's cache (cold) | a task tends to stay on one CPU, so its cache and TLB stay warm |
| load is balanced automatically | queues can become **unbalanced** (5 tasks on one CPU, 0 on another), so **load balancing** is needed (§2.11) |

### 2.8 Scheduling classes: real-time threads always go before normal ones

#### 2.8.1 The idea
Threads want different things. An audio or trading thread wants "**run me immediately, every time**". A compiler wants "**a fair share**". So Linux has several **scheduling classes**. Each class has its own rules and its own container inside the run queue, and the classes are checked in a **fixed order**:

| Order | Class | Policies (set with `chrt` / `sched_setscheduler`) | Container in the run queue | How it picks |
|---|---|---|---|---|
| 1 | **Deadline** | `SCHED_DEADLINE` | tasks sorted by deadline | earliest deadline first (EDF, note 14) |
| 2 | **Real-time (RT)** | `SCHED_FIFO`, `SCHED_RR` (priority 1–99) | 100 lists (one per priority) + a bitmap (§2.8.2) | highest priority, FIFO within a priority |
| 3 | **Fair** (normal) | `SCHED_OTHER` (the default, nice −20…+19), `SCHED_BATCH`, `SCHED_IDLE` | a red-black tree sorted by vruntime (§2.8.3) | the task that has had the least fair share (§2.9) |
| 4 | **Idle** | — | the per-CPU idle task | runs only when nothing else is runnable; puts the CPU into a low-power state |

The rule for choosing the next task on a CPU is literally this:
```c
// pick_next_task(): ask each class in order and take the first answer
for (class = deadline_class; class; class = class->next)   // deadline → rt → fair → idle
    if ((t = class->pick_next_task(rq)) != NULL)
        return t;
```
- While **any** RT task is runnable on a CPU, **no** normal task runs there. By default Linux keeps about 5 % of each second for normal tasks, so a runaway RT loop can't freeze the machine.
- Almost every thread you write lives in the fair class.
- Name-only: Linux also has an internal `stop` class above deadline, used for kernel housekeeping such as moving tasks between CPUs. Since 6.12 there is also an optional `ext` class (sched_ext), which loads schedulers written as BPF programs. (The `SCHED_IDLE` *policy* belongs to the fair class with a tiny weight. It is not the idle class.)

> **"If deadline always goes first, won't RT and fair tasks starve?"** No, for three reasons.
>
> **1. Most of the time there is no deadline or RT task at all.**
> - Every thread starts in the fair class (§2.8.4). A thread enters a higher class only if someone with privileges explicitly asks.
> - So on a normal machine the loop finds nothing in deadline or RT and falls through to fair.
> - "Deadline is checked first" means **if** a deadline task is runnable **and has budget left**, it wins.
>
> **2. Deadline tasks are budget-limited by design**, so they can't starve anyone.
> - A `SCHED_DEADLINE` task must declare `runtime` per `period`, e.g. 3 ms every 10 ms.
> - **Admission control:** `sched_setattr()` **fails** if the total of runtime/period over all deadline tasks would exceed the limit (by default 95 % of each CPU).
> - **Budget enforcement:** once a task has used its 3 ms in the current period, it is **throttled**. It stays runnable but is invisible to the deadline class until its budget refills at the next period. While it is throttled, the deadline class returns NULL, so the loop moves on to RT and fair.
> ```
>  One CPU. D = deadline task (3 ms every 10 ms), R = RT task (always runnable), F = fair task
>  t = 0–3    D runs (budget 3 → 0)          → D throttled until t = 10
>  t = 3–10   deadline class has nothing pickable → RT class → R runs
>  t = 10     D's budget refills to 3 ms, new deadline t = 20 → D preempts R
>  ⇒ D can never take more than 30 % of this CPU, however buggy it is.
> ```
>
> **3. RT tasks are capped by a reserve kept for normal tasks.**
> - RT has no budget of its own: `SCHED_FIFO` 99 in a busy loop would run forever.
> - So Linux reserves a slice for normal tasks: **at most 950 ms of every 1000 ms for RT/deadline**, set by `sched_rt_runtime_us` / `sched_rt_period_us`.
>   - Older kernels throttle the RT tasks once they have used 950 ms.
>   - Linux 6.12+ instead runs the fair class through a small **deadline server**: an internal deadline entity with 50 ms per 1 s, which therefore beats RT whenever fair tasks are being starved.
> - Result: F still gets ≈ 50 ms per second, enough for a shell to kill the runaway task.
> - HFT setups sometimes disable this (`sched_rt_runtime_us = -1`), but only on **isolated cores** where nothing else needs to run (note 13).
>
> **Inside a class:**
> - **RT:** a lower RT priority *is* allowed to starve under a higher one. That's the meaning of fixed priority, and there is no aging. Making sure every RT task still meets its deadline is the system designer's job (rate-monotonic / response-time analysis, note 14).
> - **Fair:** vruntime guarantees that nobody starves.
> - **Deadline:** admission control + EDF keeps every admitted task within its budget and deadline.

#### 2.8.2 How the RT class finds the highest priority in O(1): the bitmap
The RT container is an array of 100 linked lists plus a 100-bit **bitmap**: **bit p = 1 if and only if list p is non-empty**. Inside the kernel a lower index means a higher priority; user priority 99 maps to index 0.

Here is the idea with only 8 priorities:
```
 index :   0    1    2    3     4     5    6    7        (0 = highest priority)
 lists :  [ ]  [ ]  [A]  [ ]  [B,C]  [ ]  [ ]  [D]
 bitmap:   0    0    1    0     1     0    0    1
```
- **Pick:** take the index of the first 1-bit, which is 2 ⇒ run A. CPUs find the first set bit with a single instruction (x86 `tzcnt`/`bsf`; ARM `clz`). So picking costs the same with 3 tasks or 3,000 ⇒ **O(1)**.
- **A new task E at priority 4:** append it to list 4 (FIFO order within a priority).
- **A blocks:** remove A. List 2 is now empty ⇒ clear bit 2. The next pick is index 4 ⇒ B.
- **`SCHED_FIFO`:** B runs until it blocks, yields, or a higher-priority task appears. There is no time limit.
- **`SCHED_RR`:** the same, except that after its quantum (100 ms by default) B moves to the **back** of list 4 and C gets a turn.

Windows and the old Linux O(1) scheduler (§2.14) use the same bitmap trick.

#### 2.8.3 How the fair class finds the most deserving task: a red-black tree
- A **red-black tree** is a self-balancing binary search tree. It always stays roughly balanced, so inserting or removing a task costs **O(log n)**.
- The fair class sorts the tree by **vruntime**, roughly "how much CPU this task has used" (§2.9). Smaller vruntime = less CPU so far = more deserving.
- The **leftmost** node has the smallest vruntime. The kernel keeps a pointer to it, so "who's next?" is **O(1)**, and putting a task back is O(log n).
```
                 (vruntime 50)
                /             \
           (30)                 (70)
          /    \               /
      (12)     (41)         (60)
        ▲
        leftmost = next to run (pointer cached)
```
- Why not a sorted linked list? Inserting into one costs O(n).
- The tree also removes *any* task in O(log n), which happens every time a task blocks or moves to another CPU.

#### 2.8.4 How does a thread get its class and priority?
**Rule 1: it inherits them.** At `fork()` / `pthread_create()` the new thread copies its creator's policy, priority and nice value. The first process (systemd / init) is `SCHED_OTHER` with nice 0, so **every thread is fair-class, nice 0, unless someone changes it**. The kernel never moves a thread to another class on its own (the only exception is priority inheritance, below).

**Rule 2: someone changes it explicitly** with a system call (or a command that makes one):

| Want | System call / library call | Command line | Who may do it |
|---|---|---|---|
| lower your own priority (nice 0 → 10) | `nice()`, `setpriority()` | `nice -n 10 ./job`, `renice` | anyone |
| raise priority (nice → −5) | same | `sudo renice -n -5 -p PID` | root / `CAP_SYS_NICE`, or within `RLIMIT_NICE` |
| real-time `SCHED_FIFO` / `SCHED_RR`, priority 1–99 | `sched_setscheduler()`, `pthread_setschedparam()` | `chrt -f 80 ./app` | root / `CAP_SYS_NICE`, or within `RLIMIT_RTPRIO` |
| `SCHED_DEADLINE` (runtime, deadline, period) | `sched_setattr()` | `chrt -d --sched-runtime 3000000 --sched-period 10000000 0 ./app` | root / `CAP_SYS_NICE` only, and only if admission control accepts it |

Who does this in practice:
- **Desktop audio** (PipeWire/PulseAudio) asks the `rtkit` daemon for an RT priority.
- **Threaded interrupt handlers** in the kernel are created as `SCHED_FIFO` priority 50.
- **systemd services** can set `CPUSchedulingPolicy=fifo`.
- **HFT trading threads** are set with `chrt` or `pthread_setschedparam`.

```c
// Make the calling thread SCHED_FIFO priority 80 (fails with EPERM without CAP_SYS_NICE)
struct sched_param sp = { .sched_priority = 80 };
int err = pthread_setschedparam(pthread_self(), SCHED_FIFO, &sp);

// Make a new thread start as RT: by default it would INHERIT the creator's settings instead
pthread_attr_t a;  pthread_attr_init(&a);
pthread_attr_setinheritsched(&a, PTHREAD_EXPLICIT_SCHED);
pthread_attr_setschedpolicy(&a, SCHED_FIFO);
pthread_attr_setschedparam(&a, &sp);
pthread_create(&tid, &a, worker, NULL);
```

**Where the kernel stores it, and how the class is chosen** (`task_struct` fields):
```
 policy        SCHED_NORMAL / BATCH / IDLE / FIFO / RR / DEADLINE   ← what user space asked for
 static_prio   120 + nice           (fair tasks: 100 … 139)
 rt_priority   1 … 99               (RT tasks)
 dl            runtime, deadline, period   (deadline tasks)
 prio          the EFFECTIVE priority the scheduler uses (may be temporarily boosted, see below)
 sched_class → points to dl_sched_class / rt_sched_class / fair_sched_class
```
One kernel scale (lower number = more important) decides the class:
```
   −1        0 ………………… 99        100 ……… 120 ……… 139
 deadline   RT: prio = 99 − rt_priority   fair: 120 + nice (nice −20 … +19)
            (user 99 → 0, user 1 → 98)    (nice 0 → 120)
```
- `sched_setscheduler()` / `sched_setattr()` update these fields, set `sched_class` to match, and move the task to the right container in its run queue.
- `pick_next_task` then simply follows the `sched_class` pointer order (§2.8.1).
- The RT bitmap index (§2.8.2) is this `prio` value.

**Rule 3, the one automatic change: priority inheritance.**
- Situation: a fair-class thread holds a PI mutex (`PTHREAD_PRIO_INHERIT`; inside the kernel an `rt_mutex`) that an RT thread is waiting for.
- The kernel temporarily raises the holder's `prio`, and so its class, to the waiter's level until it releases the lock. This prevents priority inversion (note 14).

**Inheritance can be turned off:** with `SCHED_RESET_ON_FORK`, the children of an RT thread go back to `SCHED_OTHER`, so a FIFO 99 server doesn't create FIFO 99 helper processes. A deadline task cannot `fork()` at all unless this flag is set, because its CPU budget can't be duplicated.

### 2.9 The fair class step by step: CFS (2007–2023) and EEVDF (2023+)

#### 2.9.1 The goal: every runnable task gets an equal share
Imagine an *ideal* CPU that runs all n runnable tasks **at the same time**, each at 1/n speed. Over any interval, each task would get exactly 1/n of the CPU.

A real CPU runs one task at a time. **CFS (Completely Fair Scheduler)** approximates the ideal: it **keeps a score of how much CPU each task has had and always runs the one furthest behind.** That score is **vruntime** ("virtual runtime").

#### 2.9.2 vruntime when all priorities are equal
Rule: when a task runs for Δ ms, `vruntime += Δ`. The task with the smallest vruntime runs next.

Example: three CPU-bound tasks, all nice 0, each running 8 ms per turn (where 8 ms comes from is §2.9.4).

| Time (ms) | Runs | vruntime A | vruntime B | vruntime C | Next (smallest) |
|---|---|---|---|---|---|
| 0 | — | 0 | 0 | 0 | A (tie: any) |
| 0–8 | A | 8 | 0 | 0 | B |
| 8–16 | B | 8 | 8 | 0 | C |
| 16–24 | C | 8 | 8 | 8 | A |
| 24–32 | A | 16 | 8 | 8 | B … |

So far this behaves like round robin: each task gets ⅓ of the CPU.

#### 2.9.3 vruntime with different priorities: nice values become weights
Each nice value maps to a **weight**:
- nice 0 = 1024;
- each nice step changes the weight by about ×1.25: nice 5 = 335, nice −5 = 3121.

The rule becomes:

  **`vruntime += Δ × 1024 / weight`**

| Task | Weight | vruntime grows at | Result |
|---|---|---|---|
| nice 0 | 1024 | 1× real time | normal share |
| nice 5 | 335 | 1024/335 = **3.06×** real time | catches up with the others after ⅓ of the CPU time ⇒ gets **~⅓ as much CPU** |
| nice −5 | 3121 | 0.33× real time | gets **~3× as much CPU** |

Example: A (nice 0) and C (nice 5) on one CPU. Their vruntimes stay equal only when A's real time ÷ C's real time = 1024 ÷ 335. So A gets 1024/1359 = **75 %** and C gets **25 %** (this is N7).
**"Fair" in CFS means equal vruntime, not equal real time.**

#### 2.9.4 How long does a task run before a switch? (the slice)
Switching to the smallest vruntime every microsecond would be perfectly fair, but the CPU would spend all its time context-switching. So CFS uses three rules:
- **Target latency** (`sched_latency`, 24 ms on a machine with 8 or more CPUs): every runnable task should get a turn within this period.
- **Slice** of a task = target latency × (its weight ÷ total weight). Three equal tasks ⇒ 24/3 = 8 ms each.
- **Minimum granularity** (3 ms): a slice is never shorter than this. With 10 tasks, 24/10 = 2.4 ms < 3 ms, so the period stretches to 10 × 3 = 30 ms (N9).

The timer tick (§2.10.2) checks whether `curr` has used up its slice. If it has, it's time to switch.

#### 2.9.5 Why interactive tasks feel fast without any "is it interactive?" guess
An editor (vim) sleeps 99 % of the time waiting for keys. **While it sleeps, its vruntime doesn't grow, but gcc's does.** So when vim wakes up, it has the **smallest vruntime**, lands in the leftmost position, and runs almost immediately. Responsiveness falls out of fairness for free.

Two corrections stop this from being abused:

1. **A long sleeper can't bank credit.**
   - The problem: vim sleeps for an hour with vruntime 400 while the others reach 1,000,000. On waking it would run alone for a long time.
   - The fix: on wake-up, `vruntime = max(old vruntime, min_vruntime − 12 ms)`. `min_vruntime` is the smallest vruntime in this run queue (the kernel tracks it), and 12 ms is half the target latency.
   - Result: a waking task gets at most a small head start.
   - Example: min_vruntime = 1000 and vim's old vruntime = 400 ⇒ vim is placed at max(400, 988) = **988**. It is 12 ms "behind", so it runs first, but its advantage is about 12 ms, not 600 ms.
2. **New tasks start at the back.** A newly forked child starts at about min_vruntime + one slice, not at 0. Otherwise a program could fork endlessly to grab the CPU.

**Wake-up preemption.** When vim wakes, should it interrupt gcc right away or wait until gcc's slice ends?
- CFS preempts if vim's vruntime is smaller than gcc's by more than a small threshold (a few ms).
- The threshold prevents constant back-and-forth switching between two tasks.

#### 2.9.6 EEVDF (Linux 6.6, 2023): what changed
**The problem with CFS:**
- CFS answers "who has had the **least** CPU?", but not "who needs the CPU **soon**?".
- A latency-sensitive task (audio, UI) and a batch task with the same weight were treated identically.
- So CFS needed many tuning knobs and special cases around wake-ups.

**EEVDF** (Earliest Eligible Virtual Deadline First) keeps vruntime fairness and adds one idea:
1. **Eligible** = the task has *not* received more than its fair share so far (its vruntime is not ahead of the queue's average). Only eligible tasks can be picked, which preserves fairness.
2. Each task has a **virtual deadline**: when its next slice "should" finish, i.e. its vruntime + slice × 1024/weight.
3. **Pick:** among eligible tasks, the one with the **earliest virtual deadline**.

Effect:
- A task that requests a **shorter slice** gets an earlier deadline, so it is picked **sooner**.
- It runs in **smaller pieces**, so its **total share stays the same**.
- Latency and CPU share become two separate knobs.

Interview line: *"CFS runs the task with the smallest weighted vruntime, kept in an RB-tree. Since 6.6 Linux uses EEVDF: the same fairness, but among tasks that haven't over-consumed, it picks the earliest virtual deadline. A shorter requested slice means lower latency, not more CPU."* The exact eligibility/lag arithmetic is beyond interview depth.

### 2.10 When does the scheduler actually run, and what happens in a switch?

#### 2.10.1 The scheduler is not a separate program
A common misconception is that "a scheduler process runs in the background and decides". It doesn't.
- The scheduler is a **kernel function, `schedule()`**.
- It runs **on the CPU it is scheduling, inside whatever task is currently there** (in kernel mode).
- It runs only when something calls it, and there are exactly two ways that happens:

| Way | Trigger | Who calls `schedule()` |
|---|---|---|
| **A. Voluntary**: the task gives up the CPU | it **blocks**: `read()` with no data, mutex already locked, `sleep()`, waiting for the disk; also `sched_yield()` and `exit()` | the task itself, from inside that system call |
| **B. Involuntary (preemption)**: the CPU is taken away | (1) the **timer tick** finds the task's slice is used up, or (2) a **more deserving task wakes up** | the kernel, at the next **safe point**, after setting a flag (§2.10.3) |

#### 2.10.2 The timer tick: what makes preemption possible
Each core has a hardware timer (note 01 §2.2: the LAPIC timer on x86, the generic timer on ARM). It interrupts the CPU every 1–4 ms (`HZ` = 1000 or 250). Without it, a `while(1);` loop would never give the CPU back.

On every tick the kernel:
1. **charges** the running task: adds the elapsed time to its runtime and vruntime;
2. **checks** whether the task has used up its slice (§2.9.4); if so, it sets the **need-resched flag** (§2.10.3);
3. occasionally starts load balancing (§2.11).

An HFT core with `nohz_full` and only one task stops this tick entirely (note 13).

#### 2.10.3 The need-resched flag: "switch soon", not "switch now"
In way B the kernel does not switch on the spot. It sets a per-task flag, `TIF_NEED_RESCHED` ("please reschedule"), and carries on.

**Why delay?**
- The tick runs inside an **interrupt handler**.
- Switching tasks halfway through an interrupt handler, or while kernel code holds a spinlock, would leave work half-done.
- For example, a spinlock could end up held by a task that isn't running, and every other CPU wanting it would spin forever.

**Where the flag is checked: only at safe points**, where nothing is half-done:
1. **On every return from the kernel to user mode**, at the end of every interrupt, exception and system call. This is the main one.
   - Example: a user `while(1)` loop is interrupted by the tick → the tick sets the flag → on the way back to user mode the kernel sees the flag → it calls `schedule()`.
2. In a **preemptible kernel** (most desktop and phone kernels), also when kernel code releases its last lock. This lets long-running kernel work be preempted too.

The delay is tiny (microseconds), and correctness is guaranteed.

#### 2.10.4 The wake-up path: how a sleeping task gets back into a run queue
When the awaited event happens (a packet arrives, a key is pressed, a mutex is released), the kernel calls `try_to_wake_up(task)`:
1. **Remove** the task from the wait queue and set its state to runnable.
2. **Choose a CPU** for it (§2.11): usually the CPU it last ran on (cache still warm), or a nearby idle one.
3. **Enqueue** it in that CPU's run queue. In the fair class its vruntime is adjusted as in §2.9.5.
4. **Decide whether it should preempt** that CPU's current task. If yes, set need-resched on the current task.
   - If that CPU is a **different** core, also send it an **IPI** (inter-processor interrupt: one core interrupting another).
   - The IPI makes that core enter the kernel and reach a safe point quickly.

#### 2.10.5 What `schedule()` does
```c
void schedule(void) {
    prev = rq->curr;
    lock(rq->lock);
    if (prev is SLEEPING)            // way A: it blocked
        dequeue(prev);               //   take it out of the run queue (it's on a wait queue now)
    else                             // way B: preempted but still runnable
        put_prev_task(prev);         //   put it back in its class's container (e.g. the RB-tree)
    next = pick_next_task(rq);       // §2.8.1: deadline → rt → fair → idle
    clear_need_resched(prev);
    if (next != prev)
        context_switch(prev, next);
    unlock(rq->lock);
}
```
**Context switch** (note 02):
1. **`switch_mm`:** if `next` belongs to a **different process**, load its page-table base register (x86 CR3, ARM TTBR). With PCID/ASID tags the TLB doesn't need a full flush. If `next` is another thread of the **same** process, this step is skipped. That is why switching between threads is cheaper (note 03).
2. **`switch_to`:** save `prev`'s registers and kernel stack pointer in its `task_struct`, then load `next`'s. When this returns, the CPU is running `next`, continuing from wherever `next` last called `schedule()`.

#### 2.10.6 Putting it all together: gcc and vim on one CPU
Every term below has been defined above.
```
t = 0 ms     CPU 0: curr = gcc (CPU-bound). vim is SLEEPING inside read() on the terminal:
             it is on the tty's wait queue, not in any run queue.
             CPU 0's fair RB-tree: {gcc}

t = 1,2,3 ms Timer ticks. Each tick charges gcc 1 ms (vruntime += 1). Slice used up?
             It's the only runnable task, so there's nothing to switch to → back to gcc.

t = 3.2 ms   Key pressed → keyboard interrupt → the tty driver stores the key →
             wakes the tty wait queue → try_to_wake_up(vim):
               1. remove vim from the wait queue; state = RUNNABLE
               2. choose CPU 0 (vim last ran there)
               3. insert vim into CPU 0's RB-tree with vruntime = max(old, min_vruntime − 12 ms)
                  ⇒ vim is now leftmost
               4. vim is far behind gcc ⇒ set NEED_RESCHED on gcc (same CPU, so no IPI)

t = 3.2 ms   The interrupt handler finishes → on the way back to user mode (a safe point)
             the kernel sees the flag → schedule():
               put_prev_task(gcc)   → gcc goes back into the RB-tree (still runnable)
               pick_next_task()     → no deadline/RT tasks → fair class → leftmost = vim
               context_switch(gcc, vim): new page tables (different process), new registers

t = 3.25 ms  vim runs ~50 µs: handles the key, redraws, calls read() again → no more input →
             BLOCKS (way A): state = SLEEPING, joins the tty wait queue, calls schedule() →
             the RB-tree holds only gcc → context_switch(vim, gcc) → gcc continues.
```
vim responded within microseconds, and nobody told the kernel that vim is interactive. It simply had a low vruntime because it sleeps most of the time.

### 2.11 Multicore: which CPU does a task run on?
Each CPU has its own run queue, so the kernel must decide two things:
- (a) where to put a task that wakes up or is newly created;
- (b) how to fix imbalance between CPUs.

**(a) Placement on wake-up** (step 2 of §2.10.4), in order of preference:
1. the **CPU it last ran on**, if that CPU isn't too busy, because the task's data is probably still in that core's L1/L2 cache;
2. otherwise an **idle CPU that shares the same last-level cache (L3)**, which is still cache-friendly;
3. sometimes a CPU next to the **task that woke it**, because a producer waking a consumer usually shares data with it.

**(b) Load balancing:**
- **When:** periodically (started from the tick), and also whenever a CPU is about to go idle.
- **What:** that CPU finds the busiest CPU nearby and **pulls** some of its runnable tasks over.
- **In levels:** moving a task costs more the further it goes, so balancing happens level by level. These levels are called **scheduling domains**:
```
 Level           Example                         Cost of moving a task       How often balanced
 SMT siblings    the two hyper-threads of a core  ~free (they share L1/L2)    very often
 Same L3 (LLC)   cores on one chip                L1/L2 must refill           often
 NUMA            a different socket               L3 refill + far memory      rarely
```
- **Cache-hot tasks stay put:** a task that ran within the last ~0.5 ms is not moved unless the imbalance is large.
- **Affinity:** you can restrict a task to certain CPUs with `taskset`, `sched_setaffinity` or `pthread_setaffinity_np`, and the balancer respects it. HFT pins each critical thread to its own isolated core, so balancing never touches it (note 13).

### 2.12 Real-time policies and cgroups

**`SCHED_FIFO` / `SCHED_RR`** (the RT class, §2.8.2) [HFT] [TI] [QCOM]:
- Priority 1–99. An RT task always beats every normal task on its CPU.
- **FIFO:** runs until it blocks or yields, or a higher RT priority arrives.
- **RR:** the same, plus a quantum shared among tasks of equal priority.
- On multicore, the kernel tries to keep the N highest-priority RT tasks running on N CPUs. It pushes an RT task to a CPU that is running lower-priority work.
- **Danger:** a FIFO busy loop starves everything else on that CPU, apart from the ~5 % reserve.

**`SCHED_DEADLINE`:**
- A task declares "I need `runtime` ms every `period` ms", e.g. 2 ms every 10 ms.
- The kernel runs tasks **earliest deadline first** (EDF, note 14).
- It **rejects** a new task if total demand would exceed the CPU (admission control).
- It **caps** each task at its declared runtime, so a buggy task can't hurt the others.

**cgroups** [AMZN]:
- A **cgroup** (control group) is a set of processes that the kernel limits and accounts for together. Every Docker container and Kubernetes pod is one.
- For CPU, fairness becomes **two-level**: the CPU is first split between groups by `cpu.weight`, then between the tasks inside each group. Each group is a single entity in its parent's RB-tree.
- **Example:** user A runs 1 task and user B runs 9.
  - Per-task fairness: A gets 10 %.
  - One cgroup per user with equal weights: A gets **50 %** (N12).
- **Kubernetes:**
  - A CPU **request** becomes `cpu.weight`, a relative share that matters only when the CPU is contended.
  - A CPU **limit** becomes `cpu.max`, a quota per period such as 50 ms per 100 ms. Once the quota is used up, the group is **throttled** (not run) until the next period (note 15).

### 2.13 Frequency and energy: big.LITTLE / P-cores and E-cores [QCOM] [APPLE]
Phones (Snapdragon, Apple A-series) and recent laptops have two kinds of cores: **big / performance** cores (fast, power-hungry) and **LITTLE / efficiency** cores (slower, frugal). So the scheduler also decides **which kind of core** a task runs on. It also works with **DVFS** (dynamic voltage and frequency scaling, i.e. lowering clock and voltage to save power).

The main pieces:
- **Utilisation tracking:** the kernel keeps a decaying average of how busy each task has been recently. Linux calls this **PELT**.
- **schedutil:** sets each CPU's frequency from that utilisation. A busy CPU gets a higher clock.
- **EAS (Energy-Aware Scheduling):** uses a table of how much energy each core type uses at each frequency. It places a task on the core that does the work for the **least energy**: a light task goes to a little core, a heavy or urgent task to a big core.
- **uclamp:** user space can say "treat this task as at least X % busy". Android uses this to push the foreground app's UI thread onto big cores at a high frequency.

### 2.14 History: why Linux replaced its scheduler three times
Now that run queues, bitmaps and vruntime are familiar, the history is short:

| Version | Design | Problem that forced the next change |
|---|---|---|
| Origins: CTSS (1962), classic Unix | multilevel feedback queues (MLFQ, §2.4). In Unix, a task's priority drops as it uses CPU and recovers over time | Unix recomputed every process's priority each second (O(n)); weak multiprocessor support |
| **Linux 2.4** (2001) | **one global run queue** with one lock; every decision **scans every runnable task** and computes a "goodness" score | O(n) per decision (N10), and every CPU fights over one lock |
| **Linux 2.6.0** (2003), the **O(1) scheduler** | **per-CPU** run queues; 140 priority lists + a bitmap (the §2.8.2 trick) in two arrays, *active* and *expired*. A task that uses up its slice moves to *expired*; when *active* is empty, the two pointers are swapped | to look responsive it **guessed** which tasks were interactive (from how long they slept) and boosted them. The guesses were often wrong, causing stutters and unfairness, and programs could game them |
| **Linux 2.6.23** (2007), **CFS** | per-CPU **RB-tree ordered by vruntime** (§2.9). No guessing: sleepers win naturally | still needed extra knobs and special cases for latency |
| **Linux 6.6** (2023), **EEVDF** | vruntime fairness + **virtual deadlines** (§2.9.6) | current |

The O(1) scheduler's two arrays (priorities 0–99 = real-time, 100–139 = normal):
```
 ACTIVE (slice left)                     EXPIRED (slice used up)
 prio 100 → [T3]   ← first set bit ⇒ run T3
 prio 120 → [T1, T5]                     prio 120 → [T2]
 When ACTIVE becomes empty: swap(ACTIVE, EXPIRED). That's two pointers, so O(1), no recalculation loop.
```

### 2.15 Windows, macOS/iOS, Android (what those companies expect)

#### 2.15.1 Windows (the NT kernel in Windows 10/11 and Windows Server) [AMZN]
Read this after §2.7–2.10. The building blocks are the same as Linux (per-CPU queues, a bitmap, a flag-then-safe-point switch); the **policy for normal threads is different**: Linux uses fairness (vruntime), Windows uses **strict priority plus temporary boosts**.

**A. The same ideas under Windows names**

| Idea | Linux (§2.7–2.11) | Windows |
|---|---|---|
| what gets scheduled | thread (`task_struct`) | thread (`KTHREAD`); a process is only a container |
| run queue | per-CPU `rq` | per-CPU **ready queues**: 32 lists (one per priority) + a 32-bit **ready summary** bitmap (bit p = 1 ⇒ list p non-empty) |
| wait queue | wait queue of the object | wait list of a **dispatcher object** (event, mutex, semaphore, timer, thread, process) |
| scheduler code | `schedule()` (a function, not a thread) | the **dispatcher** (also kernel code, not a thread) |
| "switch soon" + safe point | `TIF_NEED_RESCHED`, checked on return to user mode / last lock released | request a **dispatch software interrupt**; it fires when the CPU's **IRQL** drops below `DISPATCH_LEVEL` (E) |
| normal threads | fair share by vruntime | **highest priority wins**, round robin within a level, temporary **boosts** |
| real-time threads | `SCHED_FIFO`/`RR` 1–99 | priorities **16–31** |
| group CPU limits | cgroups (`cpu.weight`, `cpu.max`) | **job objects** with CPU rate control (used by Windows containers) |

**B. Thread states (simplified)**
```
 READY ──(dispatcher selects it for a CPU)──► STANDBY ──(context switch)──► RUNNING
   ▲     (in a ready queue)                    (chosen to run next                │
   │                                            on one specific CPU)              │ waits on an object
   └──────────── object signalled (wake-up, maybe with a boost, G) ◄──── WAITING ◄┘
```
(Windows also has "deferred ready", an internal step on multiprocessors while the target CPU is being chosen. It's name-only.)

**C. The 32 priority levels**
```
 31 ┐  REAL-TIME range 16–31: fixed. The system never boosts or decays these.
    │  Needs admin privilege. Used for audio (via MMCSS, H) and some system threads.
 16 ┘
 15 ┐  DYNAMIC (variable) range 1–15: every normal app lives here.
    │  The system boosts and decays these (G), but never pushes them above 15.
  1 ┘
  0    zero-page thread (zeroes freed memory pages)
       idle thread: one per CPU, outside the scale; runs only when nothing is ready
```

**D. How a thread gets its priority: process class + thread offset**
- A thread's **base priority** = its process's **priority class** + its own **relative priority**.
- **Current priority** = base + any temporary boost (G). Boosts apply only in the 1–15 range.

| Process priority class | Base value | Thread relative priority | Effect |
|---|---|---|---|
| Idle | 4 | Idle | → 1 (or 16 in the Realtime class) |
| Below normal | 6 | Lowest | −2 |
| **Normal** (default) | **8** | Below normal | −1 |
| Above normal | 10 | **Normal** (default) | 0 |
| High | 13 | Above normal | +1 |
| Realtime | 24 | Highest | +2 |
| | | Time-critical | → 15 (or 31 in the Realtime class) |

Examples:
- Normal class + Above normal = **9**.
- High class + Highest = **15**.
- Realtime class + Time-critical = **31**.
- Normal class + Time-critical = **15**: it saturates at the top of the dynamic range and never reaches real-time.

How priorities are set and inherited:
- **Default:** a new process gets the **Normal** class. The exception is a creator in the Idle or Below-normal class; then the child inherits that class. A new thread starts at relative **Normal**.
- **Changing it:** `SetPriorityClass()` / `SetThreadPriority()`, Task Manager → *Set priority*, or `start /high app.exe`.
- **Realtime needs a privilege:** requesting the Realtime class without the "increase scheduling priority" privilege (admin) silently gives you **High** instead.
```c
SetPriorityClass(GetCurrentProcess(), HIGH_PRIORITY_CLASS);       // process base 13
SetThreadPriority(GetCurrentThread(), THREAD_PRIORITY_HIGHEST);   // this thread: 13 + 2 = 15
```

**E. When the dispatcher runs, and Windows' version of the "safe point" (IRQL)**

*Triggers* (the same as Linux §2.10.1):
- the running thread **waits** on an object (voluntary);
- its **quantum ends** (F);
- a **higher-priority thread becomes ready**, e.g. a wait is satisfied;
- a priority or affinity change, or the thread exits.

*IRQL (Interrupt Request Level)* is a per-CPU number saying what may interrupt the code that's running now:
- 0 = PASSIVE: normal thread code;
- 2 = DISPATCH_LEVEL: code holding a spinlock, or running a **DPC** (Deferred Procedure Call, the "second half" of a driver's interrupt work);
- 3 and higher: device interrupt handlers.

*Code at DISPATCH_LEVEL or higher can't be switched away from.* So the kernel never switches immediately:
1. It **requests a dispatch software interrupt**.
2. The CPU delivers it as soon as the IRQL drops below 2, i.e. when the spinlock is released or the DPC finishes.
3. Then the dispatcher runs.

This is the same idea as Linux's need-resched flag plus "no switch while `preempt_count` > 0" (§2.10.3).

**F. The quantum (time slice)**
- **Client Windows:** about 2 clock ticks, ≈ 31 ms with the default 15.6 ms tick.
- **Server Windows:** about 12 ticks, ≈ 188 ms. Longer slices mean fewer switches and better throughput for server workloads.
- **Foreground stretching (client):** threads of the process that owns the foreground window get up to **3×** the quantum. This is the setting *"Adjust for best performance of: Programs"*.
- Since Vista the quantum is charged in **CPU cycles actually used**, so a thread isn't charged for time the CPU spent handling interrupts. (Name-only.)

**G. Picking, preemption and boosts: how Windows stays responsive without vruntime**

*Pick:* find the first set bit in the 32-bit ready summary ⇒ the highest non-empty priority ⇒ take the head of that list. That is O(1), exactly the §2.8.2 trick.

*Preemption is immediate:*
- When a thread with **higher** priority than the running one becomes ready, the running thread is preempted **at once**, not at the end of its quantum.
- The preempted thread goes to the **front** of its list and keeps its remaining quantum.
- A thread whose quantum **expired** goes to the **back** of its list (round robin).

*Starvation is possible.* There is no fairness between levels: a CPU-bound thread at 11 starves a thread at 10 indefinitely. Two mechanisms make this work in practice:

**(1) Temporary boosts.** When a thread's wait ends, the code that woke it adds a boost to its current priority:

| Event that ended the wait | Boost |
|---|---|
| disk / CD / video I/O completed | +1 |
| network, named pipe, serial I/O completed | +2 |
| **keyboard / mouse** input | **+6** |
| sound I/O completed | +8 |
| event / semaphore signalled | +1 (typical) |
| a foreground-process thread finishing a wait | +2 |
| a GUI thread woken by a window message | +2 |

Rules for boosts:
- Current priority = min(base + boost, **15**). A boost never moves a thread into the real-time range, and real-time threads are never boosted.
- **Decay:** each time the boosted thread uses a full quantum, its priority drops **one level**, down to its base (N11).
- Effect: a thread that just got input runs quickly; if it then turns CPU-hungry, it sinks back. This is MLFQ-like behaviour (§2.4).

**(2) The anti-starvation safety net.**
- The **balance set manager**, a system thread, wakes once per second.
- It finds ready threads that haven't run for **~4 s** and boosts them to **15** for one quantum. Afterwards they drop straight back to their old priority.
- It's deliberately crude: each pass scans only a limited number of threads.
- It also rescues priority-inversion cases: a low-priority lock holder eventually gets to run.

**AutoBoost (Windows 8+)** is real priority inheritance for kernel locks. The lock owner is temporarily raised to the priority of its highest waiter (note 14).

**H. MMCSS: glitch-free audio without starving the system**
- Audio/video apps register their threads with `AvSetMmThreadCharacteristics(L"Pro Audio")`.
- The Multimedia Class Scheduler Service then raises those threads into the **real-time range (~16–26)** for most of each period, and drops them to a low priority for the remainder.
- By default about **20 %** of each period is kept for everything else (the `SystemResponsiveness` setting).
- This is Windows' version of Linux's 95 % RT cap (§2.8.1 box), applied per application.

**I. Multicore: which CPU runs the thread**

Each thread has an **ideal processor**, assigned round-robin when the thread is created so that a process's threads spread out. It also remembers its **last processor** (whose cache is still warm).

When a thread becomes ready, the dispatcher:
1. **Looks for an idle CPU:** preferably the ideal processor, then the last processor, then the current one. On hyper-threaded chips it prefers a core where *both* siblings are idle.
2. **If none is idle:** if the thread has higher priority than what's running (or on standby) on its ideal processor, it **preempts** that thread, sending an IPI (§2.10.4).
3. **Otherwise:** it puts the thread in the ideal processor's ready queue. CPUs that go idle look at other CPUs' queues for work.

Controls:
- **Hard affinity:** `SetThreadAffinityMask` / `SetProcessAffinityMask`.
- **Soft hint:** `SetThreadIdealProcessor`.
- **CPU Sets** (Windows 10+): soft-reserve cores, used by games and consoles.
- **Processor groups:** an affinity mask is 64 bits, so CPUs are organised in **groups of up to 64**. A bigger machine has several groups, and since Windows 11 / Server 2022 a process spreads across groups by default.

**J. Hybrid CPUs (P-cores and E-cores, Windows 11 on Intel 12th-gen and later)**
- **Intel Thread Director** is hardware that watches each thread's instruction mix and tells Windows which core type would suit it.
- Windows combines that with the thread's **QoS level**: a foreground / high-QoS thread goes to P-cores; a background, minimised or **EcoQoS** thread goes to E-cores at a lower frequency.
- Task Manager's **Efficiency mode** = EcoQoS + Idle priority class. Apps can opt in with `SetProcessInformation(... ProcessPowerThrottling ...)`.
- This is the Windows counterpart of Linux EAS (§2.13) and Apple's QoS.

**K. Linux vs Windows in one table**

| | Linux (fair class) | Windows (dynamic range 1–15) |
|---|---|---|
| core rule | least vruntime runs (fair share) | highest priority runs; round robin within a level |
| priority input | nice → weight (a CPU-share ratio) | class + relative → priority level (strict order) |
| can a normal thread starve? | no (vruntime catches up) | yes, until the ~4 s anti-starvation boost |
| responsiveness comes from | sleepers have low vruntime (§2.9.5) | explicit boosts on wake-up, then decay |
| slice | computed from weights (target latency) | fixed quantum (client ~31 ms ×3 for foreground; server ~188 ms) |
| higher-priority wake-up | preempts if far enough behind (threshold) | always preempts immediately |
| RT protection | 95 % cap / fair deadline server | MMCSS reserve (~20 %) for multimedia threads |

**Windows interview depth:**
- **Must know:** the 32 levels and their ranges, class + relative priority, strict-priority immediate preemption, boosts and decay, the ~4 s anti-starvation boost, client vs server quantum and foreground stretching, ideal processor and affinity, and the comparison with Linux.
- **Name-only:** IRQL/DPC, MMCSS, AutoBoost, Thread Director/EcoQoS, job objects, processor groups.
- **Skip:** quantum units and cycle accounting, the deferred-ready state, the exact balance-set-manager scan limits.

#### 2.15.2 macOS / iOS [APPLE]
- Developers don't set numeric priorities. They label work with a **QoS class**: user-interactive (UI), user-initiated, default, utility or background.
- The QoS class decides priority, **P-core vs E-core**, I/O priority and timer slack.
- The scheduler (**Clutch**, extended as **Edge** on Apple silicon) chooses in three steps:
  1. among QoS levels: higher QoS first, but lower levels can't starve;
  2. then fairly **between apps**;
  3. then among the threads inside an app.
  So an app with 100 threads can't crowd out another app's UI.
- If a UI thread waits on a lock held by a background thread, the holder temporarily inherits the higher QoS (priority inheritance, note 14).

#### 2.15.3 Android [QCOM]
The Linux scheduler + EAS (§2.13), plus:
- **cpusets:** the foreground app may use the big cores; background apps get only the little ones;
- **uclamp** boosts for the UI and render threads;
- Qualcomm kernels have used **WALT**, an alternative to PELT that reacts faster to load changes.

### 2.16 Is the scheduling policy the same on every core? Can an update change it?

**"Policy" can mean three different things. Separate them first:**

| Layer | What it is | Example | Same on every core? | Who changes it, and how |
|---|---|---|---|---|
| **1. Scheduler algorithm** (code) | the kernel code that implements classes, vruntime, boosts, balancing | Linux CFS → EEVDF; the Windows dispatcher | **yes**: one kernel image runs on all cores | only a **new kernel / OS update** (or, on Linux 6.12+, a sched_ext scheduler loaded at run time; see below) |
| **2. Tunables** (knobs of that code) | parameters | RT cap 950 ms / 1 s, tick rate `HZ`, Windows quantum and foreground setting, power plan | mostly system-wide; a few per-CPU or per-group | the admin: `sysctl`, boot parameters, cgroup files; on Windows, the registry or power plan |
| **3. Per-thread policy and priority** | the class a thread is in, its priority or nice value | `SCHED_FIFO` 80, nice 5, Windows High class | **no**: set per thread, so every core runs a mix | the program or admin: `chrt`, `nice`, `pthread_setschedparam`, `SetThreadPriority` (§2.8.4, §2.15.1 D) |

So **every core runs the same rules (layer 1)**. But each core has its own run queue holding its own threads, each with its own class and priority (layer 3). So each core makes its own decisions, using the same rules.

**How cores end up behaving differently even with the same code:**
- **Isolated cores:** Linux `isolcpus=`, `nohz_full=`, or a cpuset with load balancing turned off. No balancing moves tasks onto them, and the tick stops. HFT uses this (note 13).
- **Different core types** (P/E, big.LITTLE): the algorithm is the same, but it knows each core's capacity and energy cost, so placement differs (§2.13, §2.15.1 J).
- **Affinity / CPU Sets:** these restrict which threads can appear on which core.
- [TI] [QCOM] **AMP (asymmetric multiprocessing):** many SoCs run **different operating systems on different cores**, e.g. Linux on the Cortex-A cores and an RTOS (FreeRTOS, TI-RTOS, Zephyr) on a Cortex-R/M core or a DSP. Then scheduling really does differ per core, because there are two schedulers. In **SMP** (one OS on all cores) there is one scheduler.

**Can a system update change the scheduler? Yes. It's ordinary kernel code.**
- **Linux:** install a newer kernel and reboot. For example, Ubuntu 22.04 (kernel 5.15) uses CFS and Ubuntu 24.04 (kernel 6.8) uses EEVDF: the same apps, a different scheduler. Distros and Android vendors also patch the scheduler; Qualcomm's WALT ships as a vendor kernel module.
- **Windows:** scheduler changes ship in Windows releases and updates. For example, Windows 11 added Intel Thread Director support for hybrid CPUs, and scheduling fixes for new CPUs arrive through Windows Update plus chipset drivers.
- **macOS / iOS:** Clutch arrived with macOS 11 / iOS 14, and Edge with the Apple-silicon releases.
- **Firmware matters too:** a BIOS/UEFI update can change the hints the OS relies on, such as the ACPI CPPC "preferred core" ranking (which cores boost highest). Placement can change without any OS update.
- **Embedded RTOS:** the scheduler is compiled into your firmware (FreeRTOS: `configUSE_PREEMPTION` and `configUSE_TIME_SLICING` in `FreeRTOSConfig.h`). Changing it means rebuilding and reflashing, or an OTA update.

**Changing behaviour without an update or reboot:**
- **Layers 2 and 3 change live:**
  - Linux: `sysctl kernel.sched_rt_runtime_us=…`, `chrt`, `nice`, cgroup files, `tuned` profiles;
  - Windows: power plans, Game Mode, `SetPriorityClass`, CPU Sets.
- **Linux sched_ext (6.12+)** can even swap layer 1 live.
  - Root loads a scheduler written as a BPF program, e.g. `scx_lavd` (built for gaming on the Steam Deck) or `scx_rusty`. It replaces the fair class for all tasks, or only for tasks that opt in.
  - Safety net: if it misbehaves, e.g. leaves a runnable task unscheduled for too long (≈ 30 s by default), the kernel **ejects it and falls back** to the normal fair scheduler. Unloading it also reverts.
- **Windows has no pluggable scheduler.** You can only change tunables and per-thread settings.
- **Apps can never replace the algorithm.** They set only their own threads' layer-3 attributes, and raising priority needs privilege (`CAP_SYS_NICE` or admin).

### 2.17 What to know for interviews, and what you can skip
| Must explain clearly | Name-level only | Safe to skip |
|---|---|---|
| run queue (per CPU, runnable tasks only) vs wait queue | EEVDF's eligibility + virtual-deadline idea | EEVDF lag maths, PELT formula |
| classes in order: deadline > RT > fair > idle; why higher classes don't starve lower ones (budgets, 95 % cap) | Clutch / Edge, WALT | sched_ext internals, `stop` class |
| three layers: algorithm (one kernel, all cores; changed only by updates) / tunables / per-thread policy; AMP vs SMP | sched_ext live loading with fallback; firmware CPPC hints | exact update histories |
| how a thread gets its class: inherited, changed by `nice` / `chrt` / `sched_setscheduler`, needs `CAP_SYS_NICE` | `task_struct` prio scale (−1 / 0–99 / 100–139), fair deadline server | `SCHED_RESET_ON_FORK` details |
| RT bitmap is O(1); fair class = RB-tree by vruntime with weights | scheduling-domain names (SMT / LLC / NUMA) | exact tunable values and formulas |
| why sleepers get low latency; the wake-up vruntime cap | schedutil, EAS | NUMA balancing internals, FreeBSD ULE, Solaris |
| when `schedule()` runs: blocking vs tick / wake-up → flag → safe point | | |
| the wake-up path; the two context-switch steps | | |
| per-CPU queues + load balancing + affinity | | |
| history: O(n) → O(1) → CFS → EEVDF, and why each change happened | | |
| cgroups weight vs quota [AMZN]; `SCHED_FIFO` risks [HFT] | | |
| Windows: 32 levels, class + relative, strict priority + boosts/decay, 4 s anti-starvation, quantum, ideal processor; Linux-vs-Windows contrast | IRQL/DPC, MMCSS, AutoBoost, Thread Director/EcoQoS, job objects | quantum units, deferred-ready state |

**One-paragraph answer to "how does scheduling work in a modern OS?"**
> Each CPU has its own run queue holding only runnable threads; sleeping threads sit on the wait queue of whatever they're waiting for. A run queue has one container per scheduling class, checked in order: deadline tasks (EDF), then real-time tasks (priority lists with a bitmap, O(1)), then normal tasks in a red-black tree ordered by weighted virtual runtime (CFS; since Linux 6.6, EEVDF adds virtual deadlines). The scheduler is just the kernel function `schedule()`. A thread calls it when it blocks; otherwise the timer tick or a wake-up sets a need-resched flag, and the kernel calls it at the next safe point, such as return to user mode. It puts the old thread back if it's still runnable, picks the next one, and context-switches the registers (and the page tables if it's a different process). Woken threads are placed on a cache-warm or idle nearby CPU, and load balancing pulls work between CPUs level by level (SMT, shared cache, NUMA). cgroups make fairness hierarchical, and on big.LITTLE / P-E chips, energy-aware placement and frequency scaling decide which kind of core a task runs on and how fast.

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

**N10. Why Linux 2.4's O(n) scheduler had to go (§2.14).** 1000 runnable tasks; computing one task's goodness score takes ~50 ns; 16 CPUs, each making ~1000 scheduling decisions per second.
- One decision scans every task: 1000 × 50 ns = **50 µs**.
- Decisions per second, whole machine: 16 × 1000 = 16,000.
- Time inside the scheduler: 16,000 × 50 µs = 0.8 s **per second**, all under **one global lock** ⇒ that lock is held 80 % of the time, and CPUs spend their time queueing for it.
- O(1) scheduler: a find-first-set-bit on the CPU's own queue costs a few ns, whatever the number of tasks ⇒ per-CPU queues + O(1) pick fixed both problems.

**N13. Where does a waking task land? (§2.9.5)** One CPU's run queue holds X (vruntime 500) and Y (vruntime 520), so min_vruntime = 500. Target latency 24 ms ⇒ maximum head start 12 ms. Task S wakes up.

| S's old vruntime | Placed at max(old, 500 − 12) | Position | Why |
|---|---|---|---|
| 100 (slept a long time) | max(100, 488) = **488** | leftmost ⇒ runs next | long sleep is capped: it gets only a 12 ms head start |
| 495 (short nap) | max(495, 488) = **495** | leftmost, smaller lead | keeps its real (small) deficit |
| 530 (used a lot before sleeping) | max(530, 488) = **530** | after X and Y | sleeping doesn't erase CPU it already used |

**Try it (CFS weights):** One CPU, target latency 24 ms. A is nice 0 (weight 1024), B is nice −5 (weight 3121). Find each task's slice and how much each vruntime grows in one period.
<details><summary>Answer</summary>Total weight = 4145. Slice A = 24 × 1024/4145 = <b>5.93 ms</b>; slice B = 24 × 3121/4145 = <b>18.07 ms</b>. vruntime growth: A = 5.93 × 1024/1024 = 5.93; B = 18.07 × 1024/3121 = <b>5.93</b>. Equal vruntime growth, unequal real time ⇒ fair.</details>

**N11. Windows boost decay.** A Normal-class thread (base 8) wakes after keyboard input ⇒ +6 ⇒ **14**. If it keeps running, it drops one level per quantum: 13, 12, … back to **8 after 6 quanta**. A disk-I/O wake-up (+1) gives 9, back to 8 after one quantum.

**N14. Windows priorities, boosts and starvation (§2.15.1).** One CPU. All three threads are CPU-bound and always ready.
- A: Normal class (8) + Above normal (+1) ⇒ base **9**.
- B: High class (13) + Lowest (−2) ⇒ base **11**.
- C: Normal class (8) + Normal (0) ⇒ base **8**; it has just received keyboard input ⇒ current = 8 + 6 = **14**.

| Quantum | Ready priorities (A, B, C) | Runs | After its quantum |
|---|---|---|---|
| 1 | 9, 11, **14** | C | C decays to 13 |
| 2 | 9, 11, **13** | C | → 12 |
| 3 | 9, 11, **12** | C | → 11, goes to the **back** of list 11: [B, C] |
| 4 | 9, **11**, 11 | B (front of list 11) | B isn't boosted, stays 11 ⇒ back of list: [C, B] |
| 5 | 9, 11, **11** | C | C decays to 10 (its base is 8, so it can keep falling) |
| 6, 7, … | 9, **11**, 10 | B, B, B … | A (9) and C (10) **starve** |
| ~4 s later | balance set manager boosts A and C to 15 for one quantum each | A, C | then back to 9 and 10 ⇒ starving again |

So under a CPU-bound priority-11 thread, A and C get about 1 quantum (~31 ms) every ~4 s, i.e. **< 1 %** of the CPU. On Linux with nice values, every task would get a share proportional to its weight. That's the key policy difference.

**Try it (Windows):** what is the base priority of (a) a Below-normal class process's thread set to Highest, (b) a Realtime-class thread set to Lowest, (c) a High-class thread set to Time-critical? Which of them can ever be boosted?
<details><summary>Answer</summary>(a) 6 + 2 = <b>8</b>; (b) 24 − 2 = <b>22</b>; (c) Time-critical in a non-Realtime class = <b>15</b>. Only (a) and (c) are in the dynamic range 1–15, so only they can be boosted. (c) is already at 15, so a boost changes nothing (the cap is 15). (b) is real-time and is never boosted or decayed.</details>

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
- **Q: What is a run queue? Is a sleeping thread in it?** A per-CPU structure holding the **runnable** tasks (plus a pointer to the running one), with one container per scheduling class. A sleeping thread is **not** in it; it's on the wait queue of the socket/futex/disk request it's waiting for (§2.7.3–2.7.4). Follow-up: *"So what does 10,000 idle threads cost the scheduler?"* Nothing per decision, only memory.
- **Q: Is the scheduler a process or thread that runs in the background?** No. It's the kernel function `schedule()`, which runs on that CPU in the context of the current task. It's called either by the task itself when it blocks, or by the kernel at a safe point after the tick or a wake-up set the need-resched flag (§2.10.1).
- **Q: Why doesn't the timer interrupt switch tasks immediately?** It's in an interrupt handler, possibly with kernel locks held, and switching there would leave work half-done (e.g. a spinlock held by a task that isn't running). So it only sets `TIF_NEED_RESCHED`, and the switch happens at the next safe point: return to user mode, or a lock release in a preemptible kernel (§2.10.3).
- **Q: A thread is blocked in `read()` on a socket. Walk me through how it gets the CPU back when a packet arrives.** NIC interrupt → the kernel's network code processes the packet → data queued on the socket → wake the socket's wait queue → `try_to_wake_up`: remove from the wait queue, choose a CPU (cache-warm or idle nearby), enqueue in that CPU's run queue, decide whether to preempt the current task (set need-resched, IPI if it's another core) → at the next safe point `schedule()` picks it → context switch → `read()` returns the data (§2.10.4, §2.10.6).
- **Q: When exactly does Linux switch tasks?** (A) When the task blocks, it calls `schedule()` itself. (B) When the tick (slice used up) or a wake-up (more deserving task) sets need-resched, the switch happens at the next safe point (§2.10).
- **Q: Why did Linux go from O(n) → O(1) → CFS → EEVDF?**
  - The O(n) scan with a global lock didn't scale ⇒ per-CPU queues with a bitmap pick (O(1)).
  - O(1) relied on wrong-prone "is it interactive?" guesses ⇒ CFS modelled ideal fairness with vruntime, so sleepers win naturally.
  - CFS couldn't express "needs the CPU soon" ⇒ EEVDF added virtual deadlines and requestable slices (§2.14).
- **Q: The deadline class is always checked first. Why doesn't it starve RT and normal tasks?**
  - Deadline tasks are rare and must be requested explicitly.
  - Each one declares runtime/period, and admission control rejects new ones beyond ~95 % of the CPU.
  - A task that uses up its runtime is throttled until the next period, so the class returns nothing and lower classes run.
  - RT tasks have no budget, so the kernel keeps ~50 ms per second for normal tasks (RT throttling, or the fair deadline server in 6.12+).
  - Follow-up: *"Can a low RT priority starve under a higher one?"* Yes, by design; schedulability analysis is the designer's job (§2.8.1 box).
- **Q: How does a thread end up in a particular class and priority?** It inherits its creator's policy, priority and nice value (so by default everything is `SCHED_OTHER`, nice 0). It changes only through explicit calls: `nice`/`setpriority`, `sched_setscheduler`/`pthread_setschedparam` (FIFO/RR, needs `CAP_SYS_NICE` or `RLIMIT_RTPRIO`), or `sched_setattr` (DEADLINE). The kernel's only automatic change is a temporary priority-inheritance boost (§2.8.4). Follow-up: *"How do you stop children of an RT process inheriting RT?"* `SCHED_RESET_ON_FORK`.
- **Q: Is the scheduling policy per core, per process, or system-wide?**
  - The **algorithm** is system-wide: one kernel, the same code on every core, changed only by a kernel/OS update.
  - **Policy and priority** are per thread: `SCHED_FIFO`, nice, Windows class + relative priority. So each core runs a mix.
  - **Tunables** are mostly system-wide, with some per-CPU (isolated cores) and per-group (cgroups) settings.
  - Exception: AMP SoCs run a different OS, and so a different scheduler, on some cores (§2.16).
  - Follow-up: *"How would you try a custom scheduler on a Linux server without rebooting?"* Load a sched_ext BPF scheduler (6.12+); the kernel falls back to the fair class if it stalls tasks.
- **Q: Why is Linux's RT class still bitmap-based (O(1)) when normal tasks use an RB-tree?** Strict fixed priority only needs "the highest non-empty level", and find-first-set-bit does that in one instruction. Fairness isn't needed, so no vruntime ordering is required (§2.8.2).
- **Q: How does Linux decide which CPU a woken thread runs on?** The CPU it last ran on (cache still warm), else an idle CPU sharing the same L3, sometimes one near the waker. Load balancing (periodic, and when a CPU goes idle) then pulls work between CPUs, level by level: SMT siblings, then the same L3, then NUMA (§2.11).
- **Q: In CFS, a task slept for an hour. Does it get an hour of CPU when it wakes?** No. Its vruntime is raised to at least min_vruntime minus half the target latency (≈12 ms), so it runs soon but only gets a small head start (§2.9.5, N13).
- **Q [AMZN]: How do Kubernetes CPU requests and limits map onto the Linux scheduler?** Requests → `cpu.weight` (proportional share in the cgroup hierarchy); limits → `cpu.max` quota/period (CFS bandwidth control ⇒ throttling).
- **Q [APPLE]: How does macOS keep the UI smooth while heavy background work runs?** QoS classes (UI = user-interactive, P-cores; background = E-cores, low priority), the Clutch scheduler (choose QoS level first, then fairness between apps), and priority inheritance when the UI waits on a lock held by a background thread (§2.15).
- **Q [QCOM]: How does Android schedule on big.LITTLE Snapdragon chips?** EAS with an energy model, schedutil/WALT load tracking for frequency and placement, cpusets restricting background apps to little cores, and uclamp boosting the foreground UI and render threads.
- **Q: How does Windows keep interactive apps responsive?** Priority boosts when a wait ends (keyboard +6, network +2, disk +1, GUI message +2), decaying one level per quantum; foreground quantum stretching (up to 3×); immediate preemption by higher-priority ready threads; and an anti-starvation boost to 15 after ~4 s of waiting (§2.15.1 G).
- **Q [AMZN]: How is a Windows thread's priority decided?** Base = process priority class (Idle 4, Below normal 6, Normal 8, Above normal 10, High 13, Realtime 24) + thread relative priority (−2…+2; Idle → 1/16; Time-critical → 15/31). Current = base + boost, in the 1–15 range only. Realtime needs admin privilege, otherwise you get High (§2.15.1 D). Follow-up: *"Can a Normal-class thread reach 16 by setting Time-critical?"* No, it saturates at 15.
- **Q: Can a normal thread starve on Windows? On Linux?** Windows: yes. Strict priority means a CPU-bound thread at 11 starves everything at ≤ 10, and only the balance set manager's ~4 s boost rescues them (N14). Linux CFS/EEVDF: no; a nice +19 task still gets a weight-proportional share.
- **Q: On Windows, what happens when a higher-priority thread becomes ready while a lower one is running?** It preempts immediately (on that CPU, or on its ideal processor via an IPI). The preempted thread goes to the **front** of its ready list and keeps the rest of its quantum. The switch itself waits until the CPU's IRQL drops below DISPATCH_LEVEL (§2.15.1 E, G).
- **Q: Compare the Linux and Windows schedulers.** Both use per-CPU queues, an O(1) bitmap pick for priority levels, and deferred switching at a safe point. Linux normal tasks get a fair share by vruntime and weights. Windows uses strict priority with round robin within a level, wake-up boosts plus decay for responsiveness, a fixed quantum (client ~31 ms, server ~188 ms), and a crude 4 s anti-starvation boost (§2.15.1 K).
- **Q: How does Windows give audio threads real-time behaviour without starving the system?** MMCSS raises registered threads into the real-time range (~16–26) for most of each period and keeps ~20 % (`SystemResponsiveness`) for everyone else.
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
| Run queue vs wait queue | run queue = per-CPU, **runnable** tasks + `curr` + one container per class; wait queue = per-object list of **sleeping** tasks |
| Classes (checked in order) | deadline (EDF) > RT (`SCHED_FIFO`/`RR`, 100 lists + bitmap, O(1)) > fair (RB-tree by vruntime) > idle |
| No starvation across classes | deadline: runtime/period budget + admission control (≤ 95 %), throttled when budget is used up; RT: capped at 950 ms / 1 s (6.12+: fair deadline server 50 ms / 1 s); inside RT, lower priority may starve by design |
| Getting a class / priority | inherited at fork/pthread_create (default `SCHED_OTHER`, nice 0); changed by `nice`, `sched_setscheduler` / `pthread_setschedparam` (needs `CAP_SYS_NICE`), `sched_setattr` (DEADLINE); kernel scale −1 DL, 0–99 RT (99 − rt_prio), 100–139 fair (120 + nice); only automatic change = PI boost |
| vruntime | `+= Δ × 1024/weight`; run the smallest (leftmost); fair = equal vruntime; slice = 24 ms × w/Σw, min 3 ms |
| Sleepers | vruntime frozen while asleep ⇒ win on wake-up; capped at min_vruntime − 12 ms; new tasks start at the back |
| EEVDF (6.6+) | eligible (not ahead of fair share) + earliest virtual deadline; shorter slice = lower latency, same share |
| When it switches | (A) blocks ⇒ calls `schedule()` itself; (B) tick / wake-up ⇒ `TIF_NEED_RESCHED` ⇒ switch at a safe point (return to user mode) |
| `schedule()` | put prev back (or dequeue if sleeping) → pick_next_task → context switch (page tables if different process, then registers) |
| Wake-up path | remove from wait queue → choose CPU (last / idle sharing L3) → enqueue → maybe preempt (IPI if remote) |
| Balancing | pull from busiest CPU, periodic + when going idle; levels SMT → L3 → NUMA; cache-hot tasks stay; affinity respected |
| History | Linux 2.4 O(n) global queue → 2.6.0 O(1) per-CPU bitmaps (bad interactivity guesses) → CFS 2.6.23 (vruntime, no guessing) → EEVDF 6.6 |
| cgroups | two-level fairness by `cpu.weight` (request); `cpu.max` quota/period throttles (limit) |
| big.LITTLE | utilisation → schedutil (frequency) + EAS (least-energy core); uclamp boosts UI |
| Same on all cores? | algorithm = kernel code, same on every core, changed by OS/kernel updates (CFS → EEVDF came with kernel 6.6) or live via sched_ext; tunables mostly global; policy/priority per thread; AMP = different OS per core |
| Windows levels | 0 zero-page · 1–15 dynamic (boosted/decayed) · 16–31 real-time (fixed, admin); idle thread outside the scale |
| Windows priority | base = class (4/6/**8**/10/13/24) + relative (−2…+2, Idle → 1/16, Time-critical → 15/31); current = min(base + boost, 15) |
| Windows dispatch | per-CPU 32 ready lists + 32-bit summary bitmap; highest wins, RR within a level; immediate preemption (preempted → front, expired → back); switch when IRQL < DISPATCH_LEVEL |
| Windows responsiveness | boosts: disk +1, net +2, kbd/mouse +6, sound +8, GUI msg +2; decay 1 level per quantum; foreground quantum ×3; ~4 s starvation → 15 for one quantum; AutoBoost = PI; MMCSS for audio |
| Windows quantum / multicore | client ~2 ticks (≈31 ms), server ~12 (≈188 ms); ideal + last processor, idle core first; affinity masks, groups of 64; Thread Director + EcoQoS on P/E cores |
| macOS/iOS | QoS classes pick priority + P/E core; Clutch: QoS level → per-app fairness → threads; priority inheritance |
| Android/QCOM | EAS + schedutil/WALT, cpusets (background → little cores), uclamp |
| Example set result | WT: FCFS 8.75, SJF 7.75, SRTF 6.5, RR(4) 11.75 |
