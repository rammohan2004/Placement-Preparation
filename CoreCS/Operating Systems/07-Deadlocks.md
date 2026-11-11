# 07. Deadlocks

> **Asked at:** [AMZN] [DESHAW] four conditions, prevention vs avoidance vs detection, Banker's algorithm numericals (also OAs/GATE-style) · [HFT] [NVIDIA] [QCOM] lock ordering in real code, debugging a hung process · [TI] deadlock in RTOS tasks.
> **Prerequisites:** [05 Synchronization](05-Synchronization-Primitives.md), [06 Classic Problems](06-Classic-Synchronization-Problems.md).

---

## 1. Intuition
A 4-way intersection where four cars each enter at the same moment. Each car occupies its part of the junction and waits for the space held by the car on its right. Nobody can move, nobody will back up ⇒ **gridlock**. A **deadlock** is a set of threads where **each waits for a resource held by another in the set**, so none can ever proceed.

---

## 2. Deep Dive

### 2.1 The four necessary conditions (Coffman, 1971)
A deadlock can happen **only if all four hold at the same time**:
| # | Condition | Meaning | Intersection analogy |
|---|---|---|---|
| 1 | **Mutual exclusion** | a resource can be held by only one thread | one car per square |
| 2 | **Hold and wait** | a thread holds one resource while waiting for another | car sits in its square waiting for the next |
| 3 | **No preemption** | resources can't be forcibly taken; only released voluntarily | no tow truck |
| 4 | **Circular wait** | a cycle T1 → T2 → … → Tn → T1, each waiting for the next | the 4 cars wait in a circle |
Break **any one** ⇒ no deadlock.

### 2.2 Resource Allocation Graph (RAG)
```
   T1 ──requests──► [R1] ──assigned to──► T2
   ▲                                       │
   └──assigned── [R2] ◄──requests──────────┘
   Cycle T1 → R1 → T2 → R2 → T1  ⇒ deadlock (single-instance resources)
```
- Circles = threads, boxes = resources (dots inside = instances). Request edge T → R, assignment edge R → T.
- **Single instance per resource:** cycle ⇔ deadlock.
- **Multiple instances:** a cycle is **necessary but not sufficient** (another instance might be freed by a thread outside the cycle).
- For single-instance resources, the **wait-for graph** (only threads, T1 → T2 if T1 waits for T2) is enough: deadlock ⇔ cycle. Detecting a cycle = DFS, O(V + E).

### 2.3 Four strategies
| Strategy | Idea | Cost | Used where |
|---|---|---|---|
| **Prevention** | design so one of the 4 conditions can **never** hold | restrictive, can lower utilisation | **real code: lock ordering** |
| **Avoidance** | before granting a request, check the system stays in a **safe state** (Banker's) | needs max demands in advance; runtime checks | theory, some embedded/resource managers |
| **Detection + recovery** | let deadlocks happen, detect cycles periodically, then kill/rollback | detection cost + losing work | databases (kill a transaction), lock-order checkers |
| **Ignore (ostrich)** | assume it's rare; reboot / kill if it happens | none | general-purpose OSes for user resources (Linux, Windows) |

### 2.4 Prevention: breaking each condition
| Condition | How to break it | Practical? |
|---|---|---|
| Mutual exclusion | make resources shareable (read-only data, lock-free structures) | only sometimes |
| Hold and wait | request **all** resources at once (`std::scoped_lock(m1, m2)`), or release everything before requesting more | yes for small sets; may waste resources |
| No preemption | if a request fails, **release what you hold** and retry (`try_lock` + back off) | yes, but risk of **livelock** |
| Circular wait | impose a **global order** on resources; always acquire in increasing order | ✅ **the standard technique** (Linux kernel lock ordering, `lockdep`) |

### 2.5 Avoidance: safe state and the Banker's algorithm
**Safe state:** there's **some order** (a *safe sequence*) in which every thread can get its maximum remaining need, finish, and release everything. Safe ⇒ no deadlock is possible. Unsafe ⇒ deadlock *may* happen (not certain).

**Data** (n threads, m resource types):
- `Available[m]` = free instances now
- `Max[n][m]` = maximum each thread may ever request
- `Allocation[n][m]` = currently held
- `Need = Max − Allocation`

**Safety algorithm:**
```
Work = Available; Finish[i] = false for all i
repeat: find i with Finish[i] == false and Need[i] ≤ Work
        Work = Work + Allocation[i]     (i finishes and releases everything)
        Finish[i] = true
until no such i
Safe ⇔ all Finish[i] == true
```
**Resource-request algorithm** (thread i requests `Req`):
1. If `Req > Need[i]` → error (it exceeded its declared max).
2. If `Req > Available` → wait.
3. **Pretend** to grant: `Available −= Req; Allocation[i] += Req; Need[i] −= Req`.
4. Run the safety algorithm. Safe ⇒ grant for real. Unsafe ⇒ roll back, thread waits.

### 2.6 Deadlocks in real code (what engineers actually do)
- **Lock ordering:** document and enforce a global order (by lock level, or by address for same-type locks).
- **`std::scoped_lock(a, b)` / `std::lock(a, b)`:** acquire several mutexes without deadlock (try-and-back-off internally).
- **Avoid calling unknown code (callbacks) while holding a lock**, since it may take other locks.
- **Timeouts:** `try_lock_for` ⇒ detect and log a suspected deadlock.
- **Tools:** Linux kernel `lockdep` (warns on lock-order inversions at run time), ThreadSanitizer / Helgrind (lock-order inversion reports), debugging a hang with `gdb -p <pid>` → `thread apply all bt` (look for threads blocked in `lock()` on each other's mutexes).
- **Databases:** detect cycles in the wait-for graph and abort one transaction (the "victim").

### 2.7 Livelock and starvation (often asked together)
| | Deadlock | Livelock | Starvation |
|---|---|---|---|
| Threads | blocked, not running | **running**, but no progress | some run, one never gets a turn |
| Example | circular wait on locks | two threads repeatedly `try_lock`, fail, release, retry in lockstep (two people stepping aside in a corridor) | low priority thread never scheduled; writer in reader-preference RW lock |
| Fix | break a Coffman condition | randomised back-off, ordering | aging, fair queues |

---

## 3. Worked Numericals

**N1. Banker's algorithm (classic).** 5 threads, resources A = 10, B = 5, C = 7.
| | Allocation (A B C) | Max (A B C) | Need = Max − Alloc |
|---|---|---|---|
| T0 | 0 1 0 | 7 5 3 | 7 4 3 |
| T1 | 2 0 0 | 3 2 2 | 1 2 2 |
| T2 | 3 0 2 | 9 0 2 | 6 0 0 |
| T3 | 2 1 1 | 2 2 2 | 0 1 1 |
| T4 | 0 0 2 | 4 3 3 | 4 3 1 |
Allocated total = (7, 2, 5) ⇒ **Available = (10, 5, 7) − (7, 2, 5) = (3, 3, 2)**.

Safety check (scan T0→T4 repeatedly):
| Step | Work before | Pick (Need ≤ Work?) | Work after (+Alloc) |
|---|---|---|---|
| 1 | (3,3,2) | T0 (7,4,3)? no · **T1 (1,2,2)** ✓ | (5,3,2) |
| 2 | (5,3,2) | T2 (6,0,0)? no · **T3 (0,1,1)** ✓ | (7,4,3) |
| 3 | (7,4,3) | **T4 (4,3,1)** ✓ | (7,4,5) |
| 4 | (7,4,5) | **T0 (7,4,3)** ✓ | (7,5,5) |
| 5 | (7,5,5) | **T2 (6,0,0)** ✓ | (10,5,7) = total ✓ |
Safe sequence: **⟨T1, T3, T4, T0, T2⟩** ⇒ **safe**. (Other safe sequences exist, e.g. ⟨T1, T3, T4, T2, T0⟩.)

**N2. Request by T1 = (1, 0, 2).**
1. ≤ Need(T1) = (1,2,2) ✓. 2. ≤ Available (3,3,2) ✓.
3. Pretend: Available = (2,3,0); Alloc(T1) = (3,0,2); Need(T1) = (0,2,0).
4. Safety: Work (2,3,0) → T1 (0,2,0) ✓ → (5,3,2) → T3 ✓ → (7,4,3) → T4 ✓ → (7,4,5) → T0 ✓ → (7,5,5) → T2 ✓ → (10,5,7). **Safe ⇒ grant.**

**N3. Then T0 requests (0, 2, 0).** ≤ Need(T0) (7,4,3) ✓; ≤ Available (2,3,0) ✓. Pretend: Available = (2,1,0).
Who can finish with Work (2,1,0)? T0 needs (7,2,3) ✗; T1 (0,2,0): B 2 > 1 ✗; T2 (6,0,0) ✗; T3 (0,1,1): C 1 > 0 ✗; T4 (4,3,1) ✗. Nobody ⇒ **unsafe ⇒ deny**; T0 waits. (The resources *are* free. Banker's refuses because granting could lead to deadlock.)

**N4. Minimum resources so deadlock is impossible.** n threads each need at most k instances of one resource type. Worst case: each holds k − 1 and waits for 1 more. One extra instance lets someone finish:
**R ≥ n(k − 1) + 1.** Example: 3 threads, each needs 4 ⇒ 3 × 3 + 1 = **10**.
Reverse: R = 6 tape drives, each process needs 2 ⇒ n(2 − 1) + 1 ≤ 6 ⇒ **n ≤ 5** processes guaranteed deadlock-free.
General (different maxima kᵢ): R ≥ Σ(kᵢ − 1) + 1.

**Try it:** Available = (1, 1, 2). Allocation / Max: T0 (1,0,1)/(2,1,2), T1 (1,1,0)/(2,1,1), T2 (0,1,0)/(1,2,2). Safe? Sequence?
<details><summary>Answer</summary>Need: T0 (1,1,1), T1 (1,0,1), T2 (1,1,2). Work (1,1,2): T0 ✓ → (2,1,3) → T1 ✓ → (3,2,3) → T2 ✓ → (3,3,3). <b>Safe, ⟨T0, T1, T2⟩</b>.</details>

---

## 4. Interview Questions

### Basic
- **Q: What is a deadlock? The four conditions?** §2.1.
- **Q: Deadlock vs starvation vs livelock?** §2.7.
- **Q: Does a cycle in a RAG always mean deadlock?** Only with single-instance resources; with multiple instances it's necessary, not sufficient.

### Intermediate
- **Q: Prevention vs avoidance vs detection?** §2.3.
- **Q: How do you prevent deadlock in real multithreaded code?** A global lock order (breaks circular wait), `std::scoped_lock` for multiple locks, no callbacks under locks, try-lock with timeouts.
- **Q: Explain the Banker's algorithm.** §2.5 plus a quick run of N1.
- **Q: What is a safe state? Is an unsafe state a deadlock?** Safe = a safe sequence exists. Unsafe = deadlock is *possible*, not certain.

### Advanced / killer follow-ups
- **Q: Why don't general OSes use Banker's?** Processes don't declare max needs, the number of processes/resources changes constantly, and the O(n²m) check per request is too costly ⇒ ostrich + lock ordering in the kernel.
- **Q [HFT][NVIDIA]: A production process hangs. How do you find out whether it's a deadlock?** `top` shows ~0 % CPU (deadlock) vs 100 % (livelock/spin). `gdb -p` → `thread apply all bt` → threads blocked in `pthread_mutex_lock`/`futex_wait`. Identify the owners (`print mutex.__data.__owner`), find the cycle, then fix the lock order.
- **Q: Two threads call `transfer(a, b)` and `transfer(b, a)`. Fix it.** Order the locks by account ID or address, or use `std::scoped_lock lk(a.m, b.m)`.
- **Q: Can deadlock happen with a single lock?** Yes: a thread locking a non-recursive mutex twice; a lock held while waiting on a condvar that only another thread needing the lock would signal; an ISR waiting on a lock held by the task it interrupted (single core).
- **Q: Does `try_lock` with back-off eliminate deadlock?** It breaks "no preemption" ⇒ no deadlock, but it can **livelock** without randomised back-off.

---

## 5. Common Traps
- Saying any one condition is *sufficient* for deadlock. All four are **necessary**.
- RAG cycle ⇒ deadlock with multi-instance resources (wrong).
- Unsafe = deadlocked (wrong).
- Forgetting to check `Req ≤ Need` before `Req ≤ Available` in Banker's.
- Calculating Available wrongly. Available = Total − Σ Allocation.
- Thinking timeouts "solve" deadlock. They only detect/recover.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Conditions | mutual exclusion, hold & wait, no preemption, circular wait (all needed) |
| RAG | single instance: cycle ⇔ deadlock; multi: cycle necessary only |
| Strategies | prevent / avoid (Banker's) / detect + recover / ignore |
| Practical prevention | global lock ordering; `std::scoped_lock`; no callbacks under locks |
| Banker's | Need = Max − Alloc; safety loop Work += Alloc; request: ≤Need, ≤Avail, pretend, check |
| Safe vs unsafe | safe = a safe sequence exists; unsafe ≠ deadlock |
| Min resources | R ≥ Σ(kᵢ − 1) + 1 |
| Livelock | busy but no progress; randomised back-off |
| Debug a hang | CPU ~0 %, gdb `thread apply all bt`, find the cycle |
