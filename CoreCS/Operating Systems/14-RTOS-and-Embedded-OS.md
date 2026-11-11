# 14. RTOS & Embedded OS

> **Asked at:** [TI] [QCOM] RTOS vs GPOS, task states, ISR design, priority inversion (Mars Pathfinder), RMS/EDF schedulability numericals, semaphores vs mutexes vs queues, watchdog, boot · [APPLE] firmware / always-on processors (RTOS-based) · [NVIDIA] embedded Tegra/Jetson, microcontrollers inside GPUs · [AMZN] devices (Kindle, Echo) firmware.
> **Prerequisites:** [04 Scheduling](04-CPU-Scheduling.md), [05 Synchronization](05-Synchronization-Primitives.md), [COA 13 Interrupts](../Computer%20Organization%20and%20Architecture/13-IO-Interrupts-DMA.md).

---

## 1. Intuition
A **real-time** system is judged not only on getting the right answer but on getting it **on time**. An airbag controller that computes the perfect inflation 50 ms late has failed. Real-time doesn't mean *fast*; it means **predictable**: the worst case is known and bounded.

A general-purpose OS (Linux desktop) is a restaurant optimising for **how many customers it serves per hour**. An RTOS is an operating theatre where every step has a **deadline**, and the highest-priority job **always** runs first, immediately.

---

## 2. Deep Dive

### 2.1 Hard, firm, soft real-time
| Type | Missing a deadline means | Examples |
|---|---|---|
| **Hard** | system failure (possibly catastrophic) | airbag, ABS brakes, pacemaker, motor control loop, flight control |
| **Firm** | the late result is useless, but an occasional miss is tolerable | video frame decode, radar sample, trading quote (arguably) |
| **Soft** | quality degrades gracefully | audio streaming, UI responsiveness |
Key terms:
- **Latency:** time from an event to the response.
- **Jitter:** variation in that latency.
- **Determinism:** bounded worst case.
- **WCET (worst-case execution time):** the longest a task can take.

### 2.2 RTOS vs general-purpose OS
| | RTOS (FreeRTOS, Zephyr, TI-RTOS/SYS-BIOS, ThreadX, VxWorks, QNX) | GPOS (Linux, Windows) |
|---|---|---|
| Goal | bounded worst-case latency | throughput / fairness / average latency |
| Scheduler | **fixed-priority preemptive**: the highest-priority ready task runs **immediately** | fair-share (CFS/EEVDF), priorities adjusted dynamically |
| Size | KBs of flash/RAM | MBs–GBs |
| Memory | often no MMU (MPU only); static allocation | virtual memory, paging |
| Kernel operations | constant-time / bounded (O(1) scheduler, bounded critical sections) | may take unbounded time |
| Interrupt latency | sub-µs to a few µs, bounded | variable |
| Process model | tasks (threads) in one address space, linked with the application | isolated processes |
| Examples | MCUs (Cortex-M, TI MSP430/C2000), modem DSPs [QCOM], sensor hubs | phones' application processors, servers |
Middle ground: **Linux + PREEMPT_RT** (tens of µs worst case), or a **mixed setup**: Linux on the application cores + an RTOS on a companion core (e.g. TI Sitara / Jacinto with Cortex-R/M cores, Qualcomm SoCs with DSPs/sensor cores).

### 2.3 Tasks, states and the scheduler (FreeRTOS terms)
```
          vTaskResume / event / delay expiry
 SUSPENDED ◄──────────► READY ◄──── preempted by higher priority / yield
                          │  ▲
              scheduler   │  │
                          ▼  │
                        RUNNING ──── waits on queue/semaphore/delay ───► BLOCKED (with timeout)
```
- **Fixed-priority preemptive:** whenever a higher-priority task becomes ready (e.g. an ISR gives a semaphore), it **preempts** the running task at once.
- Equal priorities: optional round-robin **time slicing** on the tick (e.g. 1 ms).
- **Idle task:** lowest priority; runs when nothing else is ready (puts the CPU to sleep: **tickless idle** saves power).
- Each task has **its own stack**, sized statically (KBs), and overflow detection (stack canary / MPU guard).

**Inter-task communication primitives:**
| Primitive | Use |
|---|---|
| **Queue** | pass data between tasks / ISR → task (copy semantics), with a blocking timeout |
| Binary semaphore | signalling, e.g. ISR → task "data ready" |
| Counting semaphore | count events or resources |
| **Mutex** | mutual exclusion with **priority inheritance** |
| Event groups | wait for a combination of bits (AND/OR) |
| Task notifications | lightweight direct-to-task signal (fastest in FreeRTOS) |

### 2.4 Real-time scheduling theory
**Periodic task model:** task τᵢ has period Tᵢ, WCET Cᵢ, and deadline Dᵢ (= Tᵢ usually). Utilisation Uᵢ = Cᵢ/Tᵢ; total U = Σ Cᵢ/Tᵢ. If U > 1, it's impossible on one core.

**Rate Monotonic Scheduling (RMS):** **static** priorities, **shorter period ⇒ higher priority**. RMS is optimal among fixed-priority schemes (when D = T).
- **Liu & Layland sufficient test:** U ≤ n(2^{1/n} − 1).
  | n | 1 | 2 | 3 | 4 | 5 | ∞ |
  |---|---|---|---|---|---|---|
  | Bound | 1.000 | 0.828 | 0.780 | 0.757 | 0.743 | ln 2 ≈ **0.693** |
- U ≤ bound ⇒ **schedulable**. U > bound ⇒ **inconclusive**, not necessarily unschedulable ⇒ use **response-time analysis**.

**Response-Time Analysis (exact test for fixed priorities):**
Rᵢ = Cᵢ + Σ_{j ∈ higher priority} ⌈Rᵢ / Tⱼ⌉ · Cⱼ. Iterate from Rᵢ = Cᵢ until it converges. Schedulable ⇔ Rᵢ ≤ Dᵢ for every task.
(⌈Rᵢ/Tⱼ⌉ = how many times higher-priority task j can preempt task i during i's response window.)

**Earliest Deadline First (EDF):** **dynamic** priority: the job with the nearest absolute deadline runs. Optimal on one core: schedulable ⇔ **U ≤ 1** (when D = T). Downsides:
- harder to implement and analyse under overload: when overloaded, **any** task may miss (domino effect);
- RMS fails predictably, with the low-priority tasks missing first.
Linux `SCHED_DEADLINE` implements EDF + CBS.

### 2.5 Priority inversion and the Mars Pathfinder bug (must know)
```
Priority: H (high) > M (medium) > L (low). H and L share a mutex.

t0  L runs, locks mutex
t1  H wakes, preempts L, tries the mutex → BLOCKS (L holds it)
t2  M wakes (doesn't need the mutex) → preempts L (M > L)
t3… M runs for a long time. L can't run ⇒ can't release the mutex ⇒ H waits for M!
    H (highest priority) is effectively blocked by M (medium): UNBOUNDED priority inversion
```
**Mars Pathfinder (1997, VxWorks):** a high-priority information-bus task shared a mutex with a low-priority meteorological task; a medium-priority communications task ran long ⇒ the high task missed its deadline ⇒ a watchdog detected it and **reset the system** repeatedly. JPL reproduced it on the ground and fixed it by **enabling priority inheritance** on that mutex (via an uploaded parameter change).

**Fixes:**
| Protocol | Idea | Notes |
|---|---|---|
| **Priority inheritance (PIP)** | while L holds a mutex that H wants, L **temporarily runs at H's priority** ⇒ M can't preempt it ⇒ L finishes the CS quickly and releases ⇒ H runs | FreeRTOS mutexes, POSIX `PTHREAD_PRIO_INHERIT`, Linux rt-mutex (futex PI) |
| **Priority ceiling (PCP / immediate ceiling)** | every mutex has a ceiling = the highest priority of any task that uses it; locking it raises the holder to the ceiling at once | also **prevents deadlock** and chained blocking; needs static knowledge |
| Disable preemption/interrupts during short CS | nothing can preempt | only for very short sections |
Note: **binary semaphores have no owner ⇒ no inheritance**. Use a **mutex** for mutual exclusion in an RTOS.

### 2.6 ISR design rules [TI][QCOM]
1. **Keep it short:** acknowledge/clear the interrupt flag, read the data register, hand off to a task, return.
2. **Never block** inside an ISR: no mutex take, no blocking queue receive, no `delay`, no `printf`/`malloc`.
3. Use the **FromISR** APIs (FreeRTOS `xQueueSendFromISR`, `xSemaphoreGiveFromISR`, `vTaskNotifyGiveFromISR`), then request a context switch on exit if a higher-priority task was woken (`portYIELD_FROM_ISR`) ⇒ **deferred interrupt processing**.
4. Variables shared between an ISR and a task: `volatile` (compiler) **and** atomic access or a critical section (disable interrupts briefly) for anything larger than a native word, or read-modify-write.
5. Configure **interrupt priorities** (ARM NVIC): higher-priority interrupts can **nest** (preempt lower ones). On Cortex-M, interrupts above `configMAX_SYSCALL_INTERRUPT_PRIORITY` can't call RTOS APIs.
6. **Interrupt latency** = hardware recognition + finishing the current instruction + context save (Cortex-M stacks 8 registers automatically, ~12 cycles) + any time interrupts are disabled by software (critical sections!) + higher-priority ISRs running.

### 2.7 Embedded system concerns
- **Memory:** static allocation (`xTaskCreateStatic`), no heap after init or a deterministic heap (FreeRTOS heap_4 with coalescing), stack high-water-mark checks, MPU regions.
- **Watchdog timer:** a hardware counter that resets the MCU unless the software "kicks" it periodically. Kick it from a low-priority/health task that checks all critical tasks are alive (not from an ISR, which would keep kicking even if tasks hang).
- **Boot & update:** reset vector → startup (clocks, copy `.data`, zero `.bss`) → `main` → create tasks → `vTaskStartScheduler`. Firmware updates: A/B partitions + a bootloader that verifies the image signature and falls back on failure.
- **Low power:** tickless idle, sleep modes (WFI), peripheral clock gating, wake on interrupt.
- **Time:** the tick (e.g. 1 kHz), software timers (run in a timer service task), hardware timers for precise periodic work.
- **Determinism killers:** dynamic allocation, unbounded loops, long critical sections, caches/branch prediction (WCET analysis is harder ⇒ some MCUs use tightly coupled memory, TCM).

---

## 3. Worked Numericals

**N1. RMS utilisation test (passes).** τ1 (C=1, T=4), τ2 (C=1, T=5), τ3 (C=2, T=10).
U = 0.25 + 0.2 + 0.2 = **0.65** ≤ 0.780 (n = 3) ⇒ **schedulable under RMS**.

**N2. RMS bound fails → response-time analysis.** τ1 (C=1, T=4), τ2 (C=2, T=6), τ3 (C=3, T=12). Priorities: τ1 > τ2 > τ3 (shorter period).
U = 1/4 + 2/6 + 3/12 = 0.25 + 0.333 + 0.25 = **0.833** > 0.780 ⇒ the bound test is **inconclusive**.
- τ1: R = 1 ≤ 4 ✓
- τ2: R⁰ = 2 → R¹ = 2 + ⌈2/4⌉·1 = 3 → R² = 2 + ⌈3/4⌉·1 = 3 ⇒ **R = 3 ≤ 6 ✓**
- τ3: R⁰ = 3
  - R¹ = 3 + ⌈3/4⌉·1 + ⌈3/6⌉·2 = 3 + 1 + 2 = 6
  - R² = 3 + ⌈6/4⌉·1 + ⌈6/6⌉·2 = 3 + 2 + 2 = 7
  - R³ = 3 + ⌈7/4⌉·1 + ⌈7/6⌉·2 = 3 + 2 + 4 = 9
  - R⁴ = 3 + ⌈9/4⌉·1 + ⌈9/6⌉·2 = 3 + 3 + 4 = 10
  - R⁵ = 3 + ⌈10/4⌉·1 + ⌈10/6⌉·2 = 3 + 3 + 4 = 10 ⇒ converged, **R = 10 ≤ 12 ✓**
⇒ **Schedulable**, even though U exceeds the Liu–Layland bound (the bound is sufficient, not necessary). Under EDF it's trivially schedulable (U ≤ 1).

**N3. Unschedulable under RMS, fine under EDF.** τ1 (C=2, T=5), τ2 (C=4, T=7). U = 0.4 + 0.571 = **0.971** > 0.828.
RTA for τ2: R⁰ = 4 → 4 + ⌈4/5⌉·2 = 6 → 4 + ⌈6/5⌉·2 = 8 → 4 + ⌈8/5⌉·2 = 8 ⇒ R = 8 > 7 ⇒ **misses its deadline under RMS**. EDF: U = 0.971 ≤ 1 ⇒ **schedulable**.

**N4. Priority inheritance timing.** L locks the mutex at t=0 and needs 2 ms more in the CS. H arrives at t=1 and needs the mutex; M (5 ms of work, no mutex) arrives at t=1.5.
- **Without PI:** L runs 0–1 (1 ms of CS done). H arrives at 1, preempts, blocks on the mutex. L resumes 1–1.5 (0.5 ms left). M arrives at 1.5, preempts L and runs until 6.5. L finishes its last 0.5 ms at 7.0 ⇒ H gets the mutex at **7.0 ms** (blocked 6 ms, and it would be longer if M ran longer: **unbounded**).
- **With PI:** at t=1, L inherits H's priority; M can't preempt; L finishes its CS at 2 ⇒ H runs at **2 ms** (blocked 1 ms, bounded by the CS length).

**N5. CPU load from an ISR.** An ADC interrupt at 50 kHz, ISR = 2 µs ⇒ 50,000 × 2 µs = 0.1 s/s = **10 %** CPU. At 400 kHz ⇒ 80 % ⇒ switch to **DMA** (the ADC fills a buffer, one interrupt per block).

**Try it:** τ1 (C=2, T=6), τ2 (C=2, T=8), τ3 (C=3, T=12). RMS bound test? If it fails, do RTA for τ3.
<details><summary>Answer</summary>U = 0.333 + 0.25 + 0.25 = 0.833 > 0.780 ⇒ inconclusive. τ2: R = 2 + ⌈2/6⌉·2 = 4 → 2 + ⌈4/6⌉·2 = 4 ≤ 8 ✓. τ3: R⁰ = 3 → 3 + ⌈3/6⌉·2 + ⌈3/8⌉·2 = 7 → 3 + ⌈7/6⌉·2 + ⌈7/8⌉·2 = 3 + 4 + 2 = 9 → 3 + ⌈9/6⌉·2 + ⌈9/8⌉·2 = 3 + 4 + 4 = 11 → 3 + ⌈11/6⌉·2 + ⌈11/8⌉·2 = 3 + 4 + 4 = 11 ⇒ R = 11 ≤ 12 ✓ <b>schedulable</b>.</details>

---

## 4. Interview Questions

### Basic
- **Q: What is an RTOS? How is it different from Linux?** §2.2: deterministic, fixed-priority preemptive, small, bounded latency.
- **Q: Hard vs soft real-time?** §2.1.
- **Q: What is a watchdog timer?** A hardware timer that resets the system unless the software refreshes it periodically ⇒ recovery from hangs.

### Intermediate
- **Q: Explain priority inversion and how to solve it.** §2.5: the H/M/L timeline, priority inheritance / ceiling, Mars Pathfinder.
- **Q: RMS vs EDF?** Static (shorter period = higher priority, U ≤ n(2^{1/n}−1) sufficient) vs dynamic (nearest deadline, U ≤ 1 exact, worse overload behaviour).
- **Q: What should and shouldn't you do in an ISR?** §2.6 rules: short, no blocking, FromISR APIs, defer to a task.
- **Q: Mutex vs binary semaphore in an RTOS?** A mutex has an owner + priority inheritance (use it for protection); a semaphore has no owner (use it for signalling, e.g. from an ISR).

### Advanced / killer follow-ups
- **Q: U = 0.83 for 3 tasks under RMS. Schedulable?** The bound test is inconclusive (0.83 > 0.78) ⇒ do response-time analysis (N2).
- **Q: Does priority inheritance solve deadlock?** No (two tasks locking two mutexes in opposite order still deadlock). The priority **ceiling** protocol prevents deadlock and chained blocking.
- **Q [TI][QCOM]: A variable shared between an ISR and the main loop is sometimes wrong. Why?** Missing `volatile` (the compiler caches it in a register), or a non-atomic read-modify-write / multi-word value torn by the interrupt ⇒ disable interrupts around the access or use atomics.
- **Q: Interrupt latency spikes occasionally. Where do you look?** Long critical sections that disable interrupts, higher-priority ISRs, flash wait states / cache misses, priority misconfiguration (NVIC), DMA bus contention.
- **Q: Where should the watchdog be kicked?** From a supervisor task that verifies every critical task reported progress, never unconditionally from a timer ISR.
- **Q: Can Linux be real-time?** With PREEMPT_RT, bounded worst-case latency of tens of µs (soft/firm RT, some hard RT use); for µs-level hard RT use an RTOS or a dedicated MCU core.

---

## 5. Common Traps
- "Real-time = fast". It means predictable/bounded.
- Treating U > Liu–Layland bound as unschedulable (it's inconclusive; do RTA).
- Using a binary semaphore for mutual exclusion in an RTOS (no priority inheritance).
- Blocking calls or `printf` in an ISR.
- Kicking the watchdog from a timer ISR.
- Forgetting the time interrupts are disabled when computing worst-case interrupt latency.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Real-time | correct **and** on time; determinism; WCET, latency, jitter |
| Hard/firm/soft | failure / useless late / degraded |
| RTOS | fixed-priority preemptive, small, static memory, bounded ops |
| Primitives | queues, binary/counting semaphores, mutex (PI), event groups, notifications |
| RMS | shorter period ⇒ higher priority; U ≤ n(2^{1/n}−1): 0.828 (2), 0.780 (3), → 0.693 |
| RTA | Rᵢ = Cᵢ + Σ ⌈Rᵢ/Tⱼ⌉Cⱼ until stable; Rᵢ ≤ Dᵢ |
| EDF | nearest deadline; U ≤ 1 exact; bad overload behaviour |
| Priority inversion | H waits on L, M preempts L; fix: inheritance / ceiling; Mars Pathfinder 1997 |
| ISR rules | short, no blocking, FromISR + deferred processing, volatile + atomic/critical section |
| Watchdog | reset if not kicked; kick from a supervisor task |
| Linux RT | PREEMPT_RT ⇒ tens of µs worst case |
