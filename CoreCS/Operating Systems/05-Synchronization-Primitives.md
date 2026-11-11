# 05. Synchronization Primitives

> **Asked at:** [AMZN] [DESHAW] race conditions, mutex vs semaphore, condition variables, critical-section requirements · [HFT] spinlock vs mutex cost, futex, seqlock, lock-free, contention · [QCOM] [TI] disabling interrupts, spinlocks in drivers, semaphores in RTOS · [NVIDIA] atomics, locks in drivers · [APPLE] os_unfair_lock, dispatch semaphores.
> **Prerequisites:** [03 Threads](03-Threads.md), [COA 10 Coherence & Consistency](../Computer%20Organization%20and%20Architecture/10-Cache-Coherence-and-Memory-Consistency.md) (atomics, fences, TTAS; read §2.7 and §3.5).

---

## 1. Intuition
Shared data is a **single-occupancy bathroom**.
- A **mutex** is the **key**: whoever holds it goes in; others wait. Only the holder returns it.
- A **spinlock** is waiting **at the door, trying the handle constantly**: wasteful if the wait is long, fastest if it's short.
- A **semaphore** is a **bowl of N tokens** (e.g. 3 parking spots): take a token to enter, put one back when you leave. Anyone can put a token back.
- A **condition variable** is a **waiting room with a bell**: "wake me when the coffee is ready". You wait without burning CPU, and someone rings when the condition might be true.

---

## 2. Deep Dive

### 2.1 The race condition
`count++` is three machine steps (load, add, store), and two threads can interleave them ⇒ **lost updates** (trace in [COA 10 §2.7.1](../Computer%20Organization%20and%20Architecture/10-Cache-Coherence-and-Memory-Consistency.md)).
- **Race condition:** the result depends on the timing/interleaving of threads.
- **Data race (C++ term):** two threads access the same location, at least one writes, and there's no synchronization ⇒ **undefined behaviour**.
- **Critical section (CS):** the code that touches shared data and must not run in two threads at once.

### 2.2 Requirements of a correct critical-section solution
1. **Mutual exclusion:** at most one thread inside the CS.
2. **Progress:** if no one is in the CS, a thread that wants in must eventually get in (the decision can't be postponed forever by threads not interested).
3. **Bounded waiting:** there's a limit on how many times others can enter before a waiting thread gets its turn (no starvation).
(Plus: no assumptions about relative speeds or number of CPUs.)

### 2.3 Peterson's algorithm (2 threads, software only)
```c
bool flag[2] = {false, false};   // flag[i]: thread i wants to enter
int  turn;                       // whose turn it is to wait

void lock(int i) {               // i = 0 or 1, j = other
    int j = 1 - i;
    flag[i] = true;              // I want to enter
    turn = j;                    // but you go first if you also want to
    while (flag[j] && turn == j) ; // wait while you want in AND it's your turn
}
void unlock(int i) { flag[i] = false; }
```
**Why it works:** if both threads want in, both set `turn`, and **whoever wrote `turn` last waits** (`turn` can only hold one value) ⇒ mutual exclusion. When the other leaves, it clears its flag ⇒ progress. Each yields at most once ⇒ bounded waiting.

**Why it breaks on real hardware:** the store `flag[i] = true` can sit in the **store buffer** while the load of `flag[j]` reads 0. That's the **Store→Load reordering** allowed even on x86 ([COA 10 §3.3](../Computer%20Organization%20and%20Architecture/10-Cache-Coherence-and-Memory-Consistency.md)) ⇒ both threads enter. Fix: make the variables `std::atomic` with `seq_cst`, or put a full fence (`mfence`) between the stores and the loop. **Interview point:** Peterson is for teaching; real locks use hardware atomics.

### 2.4 Hardware support
| Primitive | Semantics (all done atomically) | Instruction |
|---|---|---|
| **Test-and-set** | `old = *p; *p = 1; return old;` | x86 `xchg`, `lock bts`; C++ `atomic_flag::test_and_set` |
| **Compare-and-swap (CAS)** | `if (*p == expected) { *p = desired; return true; } else return false;` | x86 `lock cmpxchg`, ARM `CAS` / `LDXR/STXR` |
| **Fetch-and-add** | `old = *p; *p += v; return old;` | x86 `lock xadd`, ARM `LDADD` |
| **Disable interrupts** | no preemption on **this core** | `cli` (x86), `cpsid i` (ARM) |
**Disabling interrupts** works only on a **single core** (other cores keep running) and only in kernel/bare-metal code. It's common in MCUs/RTOS for very short critical sections [TI][QCOM]. On multicore kernels: spinlock **plus** disable local interrupts (`spin_lock_irqsave`).

### 2.5 Spinlock
```cpp
class Spinlock {
    std::atomic<bool> locked{false};
public:
    void lock() {
        while (true) {
            if (!locked.exchange(true, std::memory_order_acquire)) return; // got it
            while (locked.load(std::memory_order_relaxed))                  // TTAS: spin on a read
                _mm_pause();                                                // x86 pause hint
        }
    }
    void unlock() { locked.store(false, std::memory_order_release); }
};
```
- **Busy-waits**: burns CPU while waiting, but no sleep/wake-up cost.
- Spin on a **read** (test-and-test-and-set) so waiters spin in their own cache and don't bounce the line ([COA 10 §2.7.3](../Computer%20Organization%20and%20Architecture/10-Cache-Coherence-and-Memory-Consistency.md)).
- **Use when:** the critical section is very short (≪ context-switch time), on multicore, or where sleeping isn't allowed (interrupt handlers, inside the kernel scheduler).
- **Never** on a single-core system with preemption: the spinner just burns its time slice while the lock holder can't run. (In the kernel, taking a spinlock disables preemption for this reason.)

### 2.6 Mutex (sleeping lock) and how Linux implements it: the futex

#### 2.6.1 What a mutex must do, and why it needs the kernel
- **lock():** if the lock is free, take it. If it's taken, **sleep** until it's released. Sleeping means not spinning and using no CPU.
- **unlock():** release the lock, and if someone is sleeping on it, **wake one** of them.

Sleeping and waking can only be done by the **kernel**. Only the kernel can take a thread off its CPU's run queue, park it on a **wait queue**, and later put it back (note 04 §2.7.3–2.7.4). So a mutex needs *some* help from the kernel.

#### 2.6.2 The naive design and its problem
The naive design keeps the whole lock inside the kernel, so every `lock()` and `unlock()` is a system call.
- A system call costs ~100 ns–1 µs (mode switch, kernel entry, mitigations), and that cost is paid **even when nobody else wants the lock**.
- Real programs show that **most lock operations are uncontended**: the lock is free when you ask for it.
- So the naive design pays the kernel cost all the time, to handle a case (somebody waiting) that is rare.

How much that costs: 10 million lock+unlock pairs.
- Naive: 2 syscalls × ~200 ns × 10⁷ = **4 s**.
- Futex fast path (below): 2 atomic instructions ≈ 25 ns per pair × 10⁷ = **0.25 s**.

#### 2.6.3 The futex idea: the lock lives in user memory, the kernel only does sleep/wake
**Futex** = "**f**ast **u**serspace mu**tex**". It splits the work in two:

| Part | Where | Who touches it |
|---|---|---|
| the **lock word**: a plain `int` in the process's memory (inside `pthread_mutex_t`) | user space | threads change it directly with **atomic instructions** (CAS, exchange, §2.4). No kernel involved |
| sleeping and waking | kernel | only through the `futex()` system call, and **only when there is contention** |

The `futex()` system call has two basic operations:
```
futex(FUTEX_WAIT, addr, expected)   "If *addr is STILL equal to expected, put me to sleep on the
                                     wait queue for addr. Otherwise return immediately (EAGAIN)."
futex(FUTEX_WAKE, addr, n)          "Wake up to n threads sleeping on addr."
```
- The kernel keeps a hash table: **user address → wait queue**.
- The kernel knows nothing about "locks", "owners" or "locked/unlocked". To it, a futex is just **a wait queue identified by the address of an integer in user memory**. All the lock logic lives in user space (glibc).
- That is the whole definition of a futex.

#### 2.6.4 The lock word has three values
| Value | Meaning |
|---|---|
| **0** | unlocked |
| **1** | locked, **nobody is waiting** |
| **2** | locked, **someone may be sleeping** in the kernel |

Why separate 1 and 2? So that `unlock()` can tell whether it needs the expensive `FUTEX_WAKE` system call. If the old value was 1, nobody is asleep, so it skips the syscall.

#### 2.6.5 The code (simplified from glibc / Drepper's "Futexes Are Tricky")
```c
#include <stdatomic.h>
atomic_int word = 0;                       // the lock word (lives in user memory)

void lock(void) {
    int c = 0;
    if (atomic_compare_exchange_strong(&word, &c, 1))   // CAS: if word==0, set it to 1
        return;                                         // FAST PATH: got it, no syscall
    // SLOW PATH: lock is held (c now holds the value we saw: 1 or 2)
    if (c != 2)
        c = atomic_exchange(&word, 2);      // mark "someone is waiting"; c = old value
    while (c != 0) {                        // old value 0 ⇒ it was free ⇒ we now own it
        futex_wait(&word, 2);               // sleep, but only if word is still 2
        c = atomic_exchange(&word, 2);      // woke up: try to take it (keep the "waiting" mark)
    }
}

void unlock(void) {
    if (atomic_exchange(&word, 0) == 2)     // release; was anyone possibly asleep?
        futex_wake(&word, 1);               // yes ⇒ ask the kernel to wake one
}                                           // old value 1 ⇒ nobody asleep ⇒ no syscall
// futex_wait/futex_wake = syscall(SYS_futex, &word, FUTEX_WAIT_PRIVATE / FUTEX_WAKE_PRIVATE, val, ...)
```
**Why does the slow path always write 2, even when taking the lock?** A woken thread can't know whether others are still asleep. So it conservatively leaves the word at 2. The worst case is one unnecessary `FUTEX_WAKE` at unlock that wakes nobody: slower, but never wrong.

#### 2.6.6 Walk-through: uncontended, then contended
**Uncontended** (the common case):
```
word 0 → T1 lock():   CAS 0→1 succeeds                        → word 1   (no syscall)
         T1 unlock(): exchange(word, 0) returns 1 ⇒ no waiters → word 0   (no syscall)
Total: two atomic instructions ≈ 20–50 ns.
```
**Contended:**
```
word  event
 0    start
 1    T1 lock(): CAS 0→1 succeeds (fast path)
 1    T2 lock(): CAS fails (sees 1) → exchange(word, 2) returns 1 (≠ 0, still held)
 2    T2: futex_wait(&word, 2) → kernel checks word == 2 ✔ → T2 is put to SLEEP
          on the wait queue for &word (dequeued from its run queue; uses no CPU)
 …    T1 runs its critical section
 0    T1 unlock(): exchange(word, 0) returns 2 ⇒ futex_wake(&word, 1)
          → kernel moves T2 back to a run queue (note 04 §2.10.4)
 2    T2 runs again: exchange(word, 2) returns 0 ⇒ loop ends ⇒ T2 OWNS the lock
 0    T2 unlock(): exchange returns 2 ⇒ futex_wake → nobody sleeping → harmless extra syscall
```

#### 2.6.7 Why `FUTEX_WAIT` takes an "expected" value: the lost wake-up
Suppose `FUTEX_WAIT` simply meant "put me to sleep". Then this can happen:
```
T2: sees word == 2, decides to sleep … (about to enter the kernel)
T1:                                     unlock: word = 0, futex_wake → nobody is asleep yet → no effect
T2: … enters the kernel and sleeps
⇒ T2 sleeps FOREVER although the lock is free  (a "lost wake-up")
```
With `FUTEX_WAIT(&word, 2)` the race can't happen:
- The kernel takes the lock of that address's wait-queue bucket, **re-reads `word`**, and only goes to sleep if it is still 2.
- `FUTEX_WAKE` takes the **same** bucket lock.
- So "check, then sleep" and "wake" can't interleave. In the race above, T2's check finds 0, the kernel returns immediately, and T2 loops and takes the lock.

This "sleep only if the value is still X" rule is the core trick of the futex. The note's one-line summary, "the kernel re-checks word == 2 atomically", refers to this.

#### 2.6.8 Costs, adaptive spinning, and where futexes are used
- Uncontended lock+unlock ≈ **20–50 ns** (two atomics). Contended ⇒ syscall + sleep + wake + 2 context switches ≈ **a few µs**.
- **Adaptive mutexes** spin briefly (a few hundred iterations, re-checking the word) **before** calling `FUTEX_WAIT`. If the holder is running on another core and releases soon, you avoid the µs sleep/wake. If not, you sleep. This gets the best of spinlocks and mutexes. Examples: glibc `PTHREAD_MUTEX_ADAPTIVE_NP`; the kernel's own mutexes spin while the owner is running.
- **Built on futexes in Linux:**
  - `pthread_mutex_t` (and so `std::mutex`);
  - condition variables, `sem_t`, `pthread_join`;
  - C++20 `std::atomic<T>::wait()` / `notify_one()`, which is almost a raw futex.
- **Priority-inheritance futex** (`FUTEX_LOCK_PI`, used by `PTHREAD_PRIO_INHERIT` mutexes): the lock word stores the **owner's thread ID**, so the kernel knows whom to boost (note 04 §2.8.4, note 14).
- **The same idea on other OSes:**
  - Windows `WaitOnAddress` / `WakeByAddressSingle`, plus a user-space fast path in `SRWLOCK` and `CRITICAL_SECTION`;
  - macOS `os_unfair_lock` (the `__ulock_wait` syscall).

#### 2.6.9 Ownership and recursion
- **Ownership:** only the thread that locked a mutex may unlock it (error-checking mutexes enforce this). **Recursive mutex:** the same thread may lock it again (counts); usually a design smell.

**Spinlock vs mutex:**
| | Spinlock | Mutex |
|---|---|---|
| Waiting | busy loop (CPU 100 %) | sleep (CPU free) |
| Cost when uncontended | ~atomic op | ~atomic op (futex fast path) |
| Cost when contended | CPU burned for the wait | sleep + wake ≈ 2 context switches (~µs) |
| Best for | CS ≪ context-switch time; can't sleep (IRQ context) | longer CS, or may block inside |
| Can hold while sleeping/blocking? | ❌ never | ✅ |

### 2.7 Semaphores
An integer **S** with two atomic operations:
```
wait(S)   / P / down / sem_wait:   S = S − 1;  if S < 0 → block this thread (put it in S's queue)
signal(S) / V / up   / sem_post:   S = S + 1;  if S ≤ 0 → wake one blocked thread
```
(In this textbook version, a negative S = the number of waiting threads. POSIX semaphores never go negative; `sem_wait` blocks while the value is 0. Same behaviour.)
| Type | Initial value | Use |
|---|---|---|
| **Binary** semaphore | 1 | mutual exclusion (like a lock, but **no owner**) |
| **Counting** semaphore | N | N identical resources (connection pool, buffer slots) |
| **Signalling** semaphore | 0 | ordering: "B must run after A": A does `signal`, B does `wait` |

**Mutex vs binary semaphore (classic question):**
| | Mutex | Binary semaphore |
|---|---|---|
| Ownership | only the locker may unlock | **anyone** may signal |
| Purpose | mutual exclusion | signalling between threads (also exclusion) |
| Priority inheritance | yes (RTOS/POSIX PI mutexes) | no (no owner to boost) ⇒ priority inversion risk |
| Recursive locking | optional | no |
| Typical use | protect data | ISR → task notification, producer/consumer counts |
[TI][QCOM]: an ISR can `give` a semaphore to wake a task (an ISR must never take a mutex).

### 2.8 Condition variables
For "wait until **some condition** on shared data is true", e.g. "queue not empty".
```cpp
std::mutex m;
std::condition_variable cv;
std::queue<int> q;

void producer(int x) {
    { std::lock_guard<std::mutex> g(m); q.push(x); }
    cv.notify_one();                         // ring the bell
}
int consumer() {
    std::unique_lock<std::mutex> lk(m);
    cv.wait(lk, [&]{ return !q.empty(); }); // = while (q.empty()) cv.wait(lk);
    int x = q.front(); q.pop();
    return x;
}
```
What `cv.wait(lk)` does, **atomically**: (1) release the mutex, (2) sleep. When woken: (3) re-acquire the mutex before returning.
- **Why atomically?** If you unlocked and then slept as two separate steps, the producer could push and notify in between ⇒ you sleep forever (the **lost wake-up** problem).
- **Why a `while` loop / predicate?** (a) **Spurious wake-ups** are allowed. (b) Another consumer may have taken the item between the notify and your re-acquiring the lock. Always re-check the condition.
- `notify_one` wakes one waiter; `notify_all` wakes all (use it when several waiters might proceed, or when waiters wait for different conditions).
- A condition variable has **no memory**: a notify with no waiter is lost (unlike a semaphore, which counts). That's why the condition lives in shared data protected by the mutex.

**Monitor** = mutex + condition variables bundled with the data (Java `synchronized` methods + `wait()`/`notify()`).

### 2.9 Reader-writer locks, seqlocks, RCU
| Primitive | Idea | Readers | Writers | Use |
|---|---|---|---|---|
| **RW lock** (`std::shared_mutex`) | many readers OR one writer | take a shared lock (still writes the lock's cache line!) | exclusive | read-mostly data with longer critical sections |
| **Seqlock** [HFT] | writer increments a sequence counter before and after writing (odd = writing). Reader: read seq, read data, re-read seq; **retry if it changed or was odd** | **never block and never write** shared memory | never blocked by readers | small, frequently-read data: market-data snapshots, Linux `jiffies`/time |
| **RCU** (Linux kernel) | readers access without locks; a writer publishes a **new copy** and frees the old one after all pre-existing readers finish (grace period) | ~zero cost | copy + wait | read-mostly kernel structures (routing tables) |
Seqlock reader:
```cpp
do {
    s1 = seq.load(std::memory_order_acquire);
    copy = data;                                       // may be torn: we check below
    std::atomic_thread_fence(std::memory_order_acquire);
    s2 = seq.load(std::memory_order_relaxed);
} while (s1 != s2 || (s1 & 1));                        // retry if a writer was active
```
(Strictly, `data` should be read with relaxed atomics to avoid a C++ data race. Production seqlocks do this.)

### 2.10 Choosing and performance
- Lock **granularity**: one big lock (simple, low concurrency) vs fine-grained locks (more concurrency, deadlock risk, note 07).
- **Contention** is the real cost: a contended lock serialises threads and bounces its cache line between cores (~100 ns per handoff).
- **Lock-free** (CAS loops, SPSC ring buffers in [COA 10 §3.8](../Computer%20Organization%20and%20Architecture/10-Cache-Coherence-and-Memory-Consistency.md)): no thread can block others; harder to get right (ABA, memory reclamation).
- Progress guarantees: **blocking** (locks) < **lock-free** (someone always makes progress) < **wait-free** (everyone finishes in bounded steps).

---

## 3. Worked Numericals

**N1. Semaphore arithmetic (GATE style).** A counting semaphore starts at 10. Then 6 `wait` and 4 `signal` operations complete. Final value? 10 − 6 + 4 = **8**.
Variant: it starts at 7; 20 `wait` then 15 `signal`. Final = 7 − 20 + 15 = **2**. (With the negative-count version, after the 20 waits S = −13 ⇒ 13 threads blocked; each signal wakes one.)

**N2. Semaphore trace.** S = 2. T1 wait, T2 wait, T3 wait, T1 signal, T4 wait, T2 signal (negative-count convention).
| Op | S after | Blocked queue | Note |
|---|---|---|---|
| T1 wait | 1 | — | T1 enters |
| T2 wait | 0 | — | T2 enters |
| T3 wait | −1 | T3 | T3 blocks |
| T1 signal | 0 | — | wakes T3 (S ≤ 0) ⇒ T3 enters |
| T4 wait | −1 | T4 | blocks |
| T2 signal | 0 | — | wakes T4 |
Inside at the end: T3 and T4.

**N3. Spin or sleep?** Context switch = 3 µs, so sleep + wake ≈ 2 switches = 6 µs. Lock hold time = 0.5 µs.
Expected spin ≈ hold time ≈ 0.5 µs ≪ 6 µs ⇒ **spin**. If the hold time were 50 µs ⇒ **sleep** (spinning wastes 50 µs of CPU). Rule: spin when the expected wait < sleep/wake cost (adaptive mutexes do this automatically).

**N4. Lock contention limits speed-up.** Each operation takes 1 µs, of which 0.2 µs is inside one global lock. Max throughput regardless of the number of threads?
The lock serialises 0.2 µs per op ⇒ at most 1 / 0.2 µs = **5 M ops/s**. With 1 thread: 1 M ops/s ⇒ max speed-up = **5×** (Amdahl with serial fraction 0.2). Plus cache-line handoffs make it worse. Fix: shard the lock or go per-thread.

**Try it:** 8 threads each increment a shared counter 10⁶ times under a mutex. Uncontended lock+unlock = 25 ns, but under contention each handoff costs ~150 ns. Estimate the total time, and the time with per-thread counters summed at the end (1 ns per increment).
<details><summary>Answer</summary>All 8×10⁶ increments are serialised: 8×10⁶ × 150 ns ≈ <b>1.2 s</b>. Per-thread: 10⁶ × 1 ns = 1 ms in parallel, plus a tiny sum ⇒ ≈ <b>1 ms</b>, about 1000× faster.</details>

---

## 4. Interview Questions

### Basic
- **Q: What is a race condition? A critical section?** §2.1.
- **Q: Three requirements of a CS solution?** Mutual exclusion, progress, bounded waiting.
- **Q: Mutex vs semaphore?** §2.7 table: ownership vs counting/signalling.
- **Q: What does a semaphore's wait/signal do?** §2.7.

### Intermediate
- **Q: Spinlock vs mutex: when do you use which?** §2.6 table + N3.
- **Q: Why must `cv.wait` be in a while loop?** Spurious wake-ups, and another thread may consume the condition first.
- **Q: Why does `wait` take the mutex?** To release it and sleep atomically ⇒ no lost wake-up.
- **Q: Does Peterson's algorithm work on modern CPUs?** Not with plain variables: store→load reordering (store buffer) and compiler reordering. It needs seq_cst atomics/fences.
- **Q: How is a mutex implemented in Linux?** Futex: an atomic fast path in user space; `FUTEX_WAIT`/`FUTEX_WAKE` syscalls only when contended.
- **Q: What exactly is a futex? Does the kernel know which thread holds the lock?** It's a kernel wait queue identified by the address of an integer in user memory, plus two operations: `FUTEX_WAIT(addr, val)` (sleep if `*addr` is still `val`) and `FUTEX_WAKE(addr, n)`. The kernel doesn't know about locks or owners. The lock word (0 free / 1 locked / 2 locked with waiters) is managed in user space with atomics. Exception: PI futexes store the owner's TID so the kernel can boost the owner (§2.6.3, §2.6.8).
- **Q: Why does the lock word need a value 2 as well as 1?** So that `unlock()` makes the `FUTEX_WAKE` syscall only when someone may be asleep. Old value 1 ⇒ no syscall (§2.6.4).
- **Q: What is a lost wake-up, and how does the futex prevent it?** A waiter decides to sleep, the holder unlocks and wakes before the waiter is actually asleep, and the waiter then sleeps forever. `FUTEX_WAIT` re-checks the value under the wait-queue bucket lock that `FUTEX_WAKE` also takes, so check-then-sleep is atomic with respect to wake (§2.6.7). Follow-up: *"Where else does this pattern appear?"* Condition variables: that's why `pthread_cond_wait` takes the mutex (§2.8).

### Advanced / killer follow-ups
- **Q: Implement a semaphore using a mutex and a condition variable.**
```cpp
class Semaphore {
    std::mutex m; std::condition_variable cv; int count;
public:
    explicit Semaphore(int n) : count(n) {}
    void wait()   { std::unique_lock<std::mutex> lk(m); cv.wait(lk, [&]{ return count > 0; }); --count; }
    void signal() { { std::lock_guard<std::mutex> g(m); ++count; } cv.notify_one(); }
};
```
- **Q: Why not use a spinlock on a single-core machine?** The holder can't run while you spin ⇒ you waste the whole time slice. In the kernel, spinlocks disable preemption, and on uniprocessor builds they compile to just that.
- **Q: Why can't an interrupt handler take a mutex?** It can't sleep: there's no process context to put to sleep and resume. Use spinlocks (with interrupts disabled) or defer the work to a thread.
- **Q [HFT]: How do you share a frequently updated price snapshot with many readers without locks?** A seqlock (§2.9): readers never write shared memory, so no cache-line bouncing from readers; they retry if a writer was active.
- **Q: RW lock: why might it be slower than a plain mutex?** Every reader still atomically updates the shared reader count ⇒ cache-line ping-pong. With short critical sections that's worse than a mutex. Use seqlock/RCU for read-mostly data.
- **Q: Lock-free vs wait-free?** §2.10.

---

## 5. Common Traps
- Saying a binary semaphore and a mutex are the same (ownership, priority inheritance).
- `if` instead of `while` around `cv.wait`.
- Notifying without changing the shared state under the mutex (the waiter re-checks and sleeps again; or the notify is lost).
- Spinning with test-and-set on the lock word (cache-line storm). Use TTAS + `pause`.
- Holding a spinlock while doing something that can sleep (I/O, `malloc` in the kernel with `GFP_KERNEL`).
- Assuming Peterson/Dekker work with plain variables in C/C++.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Race / data race | timing-dependent result / unsynchronised conflicting access = UB in C++ |
| CS requirements | mutual exclusion, progress, bounded waiting |
| Peterson | flag + turn; last to write `turn` waits; needs seq_cst on real HW |
| HW atomics | TAS, CAS, fetch-add; disable IRQs (single core only) |
| Spinlock | busy-wait; short CS, can't sleep; TTAS + pause; never on uniprocessor |
| Mutex / futex | sleep; user-space CAS fast path, syscall only when contended |
| Futex details | kernel = wait queue keyed by user address; `WAIT(addr, val)` sleeps only if `*addr == val` (prevents lost wake-up); `WAKE(addr, n)`; word 0 free / 1 locked / 2 locked + waiters; unlock syscalls only if old value was 2; uncontended ≈ 20–50 ns, contended ≈ µs |
| Semaphore | counter; wait −1 (block if <0), signal +1; binary / counting / signalling; no owner |
| Condvar | wait = unlock + sleep atomically, relock on wake; while-loop; no memory |
| Seqlock | writer seq odd/even, readers retry, readers never write |
| RCU | lock-free readers, writer copies + grace period |
| Spin vs sleep | spin if expected wait < ~2 context switches |
