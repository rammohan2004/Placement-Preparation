# 06. Classic Synchronization Problems

> **Asked at:** [AMZN] [QCOM] "print odd/even with two threads", bounded buffer, implement a blocking queue (coding rounds) · [DESHAW] readers-writers starvation, dining philosophers deadlock · [HFT] producer-consumer → lock-free SPSC queue · [NVIDIA] barriers.
> **Prerequisites:** [05 Synchronization Primitives](05-Synchronization-Primitives.md).

---

## 1. Intuition
These problems are **templates**. Almost every real concurrency bug or interview coding question is one of them in disguise:
| Template | Real-life version |
|---|---|
| Producer-consumer (bounded buffer) | logging queue, network packet queue, thread-pool task queue, market-data feed handler → strategy |
| Readers-writers | configuration/cache read by many threads, updated rarely |
| Dining philosophers | any code taking **multiple locks** (bank transfer between two accounts) ⇒ deadlock |
| Ordering / alternation | "print 1 2 3 … with N threads in turn", pipelines |
| Barrier | parallel phases (simulation steps, GPU `__syncthreads()`) |

---

## 2. Deep Dive

### 2.1 Producer-consumer (bounded buffer)
A buffer with N slots. Producers add items, consumers remove them. Three rules:
1. A producer must **wait if the buffer is full**.
2. A consumer must **wait if it is empty**.
3. Only one thread modifies the buffer at a time.

**Semaphore solution (textbook):**
```
semaphore empty = N;   // free slots
semaphore full  = 0;   // filled slots
semaphore mutex = 1;   // protects the buffer

producer:                     consumer:
  item = produce();             wait(full);      // wait for an item
  wait(empty);   // a free slot wait(mutex);
  wait(mutex);                  item = remove();
  insert(item);                 signal(mutex);
  signal(mutex);                signal(empty);   // one more free slot
  signal(full);  // one more item consume(item);
```
**Why the order of the two waits matters (classic follow-up):** suppose the producer does `wait(mutex)` **before** `wait(empty)` and the buffer is full:
```
Producer: wait(mutex) ✓ (holds the lock) → wait(empty) → blocks (buffer full) …still holding mutex
Consumer: wait(full) ✓ → wait(mutex) → blocks (producer holds it)
⇒ each waits for the other: DEADLOCK
```
Rule: **wait on the counting semaphore first, take the mutex second.** The order of the signals doesn't matter.

**Condition-variable version (what you write in C++ interviews): a blocking queue**
```cpp
template<typename T>
class BoundedQueue {
    std::mutex m;
    std::condition_variable not_full, not_empty;
    std::queue<T> q;
    const size_t cap;
public:
    explicit BoundedQueue(size_t c) : cap(c) {}
    void push(T x) {
        std::unique_lock<std::mutex> lk(m);
        not_full.wait(lk, [&]{ return q.size() < cap; });
        q.push(std::move(x));
        lk.unlock();
        not_empty.notify_one();
    }
    T pop() {
        std::unique_lock<std::mutex> lk(m);
        not_empty.wait(lk, [&]{ return !q.empty(); });
        T x = std::move(q.front()); q.pop();
        lk.unlock();
        not_full.notify_one();
        return x;
    }
};
```
Two condition variables, one per condition, so a notify wakes only the kind of thread that can proceed. [HFT] With exactly **one producer and one consumer**, replace all of this with a **lock-free SPSC ring buffer** ([COA 10 §3.8](../Computer%20Organization%20and%20Architecture/10-Cache-Coherence-and-Memory-Consistency.md)): no locks, no syscalls, ~tens of ns per message.

### 2.2 Readers-writers
Many readers may read **at the same time**; a writer needs **exclusive** access.

**First readers-writers problem (reader preference):**
```
semaphore rw_mutex = 1;   // held by a writer, or by the readers as a group
semaphore mutex = 1;      // protects read_count
int read_count = 0;

writer:                          reader:
  wait(rw_mutex);                  wait(mutex);
  ...write...                      read_count++;
  signal(rw_mutex);                if (read_count == 1) wait(rw_mutex);  // first reader locks writers out
                                   signal(mutex);
                                   ...read...
                                   wait(mutex);
                                   read_count--;
                                   if (read_count == 0) signal(rw_mutex); // last reader lets writers in
                                   signal(mutex);
```
**Problem: writer starvation.** If readers keep arriving, `read_count` never reaches 0 ⇒ a writer waits forever.
| Variant | Rule | Who can starve |
|---|---|---|
| 1st (reader preference) | readers never wait unless a writer is **already writing** | writers |
| 2nd (writer preference) | once a writer is **waiting**, no new readers start | readers |
| Fair (3rd) | FIFO order of arrival (e.g. a "turnstile" semaphore everyone passes through) | nobody |
C++: `std::shared_mutex` (`lock_shared()` for readers, `lock()` for writers); fairness depends on the implementation. For hot read-mostly data, see seqlock/RCU (note 05 §2.9).

### 2.3 Dining philosophers
Five philosophers around a table, one chopstick between each pair. To eat, a philosopher needs **both** neighbouring chopsticks.
```
         P0
     c0      c1
  P4            P1
    c4        c2
      P3  c3  P2
Naive: each picks up the LEFT chopstick, then the RIGHT.
If all five pick up their left at the same moment → everyone waits for the right → DEADLOCK.
```
All four deadlock conditions hold (note 07): mutual exclusion, hold-and-wait, no preemption, circular wait.
| Solution | How | Breaks which condition |
|---|---|---|
| **Resource ordering** | number the chopsticks; always pick up the **lower-numbered first** (so P4 picks c0 before c4) | circular wait ✅ (standard real-world fix) |
| At most N−1 seated | a semaphore with value 4 lets only 4 philosophers try ⇒ at least one gets both | circular wait |
| Both or none | pick up both chopsticks atomically (inside one mutex), or `std::scoped_lock(left, right)` | hold-and-wait |
| Odd/even asymmetry | odd philosophers take left first, even take right first | circular wait |
| Waiter / arbitrator | ask a central waiter for permission | hold-and-wait |
Real code: a **bank transfer** `transfer(a, b)` locking `a` then `b`, while another thread does `transfer(b, a)` ⇒ deadlock. Fix: lock in a global order (by account ID / address), or `std::scoped_lock lk(a.m, b.m);`, which uses a deadlock-avoidance algorithm.

### 2.4 Ordering and alternation problems (coding-round favourites)
**Print odd/even with two threads alternately (1 2 3 … 10):**
```cpp
std::mutex m; std::condition_variable cv;
int n = 1; const int MAX = 10;

void printer(int parity) {                // parity 1 = odd thread, 0 = even thread
    while (true) {
        std::unique_lock<std::mutex> lk(m);
        cv.wait(lk, [&]{ return n > MAX || n % 2 == parity; });
        if (n > MAX) break;
        std::cout << n << '\n';
        ++n;
        cv.notify_all();                  // the other thread's turn
    }
}
// std::thread t1(printer, 1), t2(printer, 0); t1.join(); t2.join();
```
**Generalise to K threads printing in turn:** thread i prints when `n % K == i`. Same pattern, `notify_all`.
**With semaphores:** `sem_odd = 1, sem_even = 0`. Odd: `wait(sem_odd); print; signal(sem_even)`; even does the mirror. Each semaphore hands the "baton" to the other thread.

**"Function B must run after A finishes" (in another thread):** a semaphore initialised to 0: A ends with `signal`, B starts with `wait`. Or `std::promise`/`std::future`, or `std::latch` (C++20).

### 2.5 Barrier
N threads must all reach a point before any continues (phase-based computation).
```cpp
// C++20: std::barrier sync(N);   each thread: work_phase1(); sync.arrive_and_wait(); work_phase2();
// Hand-made reusable barrier with a generation counter:
class Barrier {
    std::mutex m; std::condition_variable cv;
    int count, waiting = 0, generation = 0; const int n;
public:
    explicit Barrier(int n_) : count(n_), n(n_) {}
    void wait() {
        std::unique_lock<std::mutex> lk(m);
        int gen = generation;
        if (++waiting == n) { waiting = 0; ++generation; cv.notify_all(); }
        else cv.wait(lk, [&]{ return gen != generation; });  // generation stops a fast thread
    }                                                        // from slipping through the next round
};
```
[NVIDIA] `__syncthreads()` is a barrier for the threads of one CUDA block.

### 2.6 Others to recognise (one line each)
- **Sleeping barber:** a bounded waiting room. Customers leave if it's full; the barber sleeps when no customers ⇒ a counting semaphore for waiting chairs + signalling semaphores.
- **Cigarette smokers:** shows that some problems can't be solved with plain semaphores without conditionals (agent/pusher pattern).
- **H₂O problem (LeetCode 1117):** 2 H threads + 1 O thread form a molecule ⇒ semaphores H=2, O=1 + a barrier of 3.

---

## 3. Worked Problems

**W1. Bounded-buffer trace.** N = 2. Sequence: P produces A, P produces B, P tries C, C1 consumes, P produces C.
| Step | empty | full | Buffer | Note |
|---|---|---|---|---|
| start | 2 | 0 | [] | |
| P: A | 1 | 1 | [A] | |
| P: B | 0 | 2 | [A,B] | |
| P: C | −1 (blocked) | 2 | [A,B] | `wait(empty)` blocks: buffer full |
| C1 consumes | 0 → wakes P | 1 | [B] | `signal(empty)` unblocks P |
| P inserts C | 0 | 2 | [B,C] | |

**W2. Spot the bug.**
```cpp
void consumer() {
    std::unique_lock<std::mutex> lk(m);
    if (q.empty()) cv.wait(lk);     // BUG
    auto x = q.front(); q.pop();
}
```
Bug: `if` instead of `while`. With two consumers, both can be woken (spurious wake-up or `notify_all`). The first takes the only item, and the second calls `front()` on an empty queue ⇒ UB/crash. Fix: `cv.wait(lk, [&]{ return !q.empty(); });`.

**W3. Readers-writers starvation timeline.** Reader-preference lock; R1 reads at t=0–10, W arrives at t=1, R2 arrives at t=5 (reads 5–15), R3 at t=12 (12–20).
`read_count` stays > 0 from t=0 to t=20 (R1 → R2 overlaps R1, R3 overlaps R2) ⇒ W can't start until **t=20**, even though it arrived at t=1. With a steady stream of readers, W starves. Writer preference would block R2 and R3 once W was waiting ⇒ W runs at t=10.

**Try it:** dining philosophers with resource ordering, 5 philosophers, chopsticks c0–c4, Pi needs ci and c(i+1 mod 5). Which philosopher picks up chopsticks in the "opposite" order from the others, and why does this prevent deadlock?
<details><summary>Answer</summary>P4 needs c4 and c0; lower-first means it picks <b>c0 first</b>, while everyone else picks their left (ci) first. A cycle would need every philosopher holding one chopstick and waiting for the next in a circle. But P0 and P4 both try c0 first, so one of them gets nothing and holds nothing, which breaks the circle. In general, a global lock order makes a cycle impossible.</details>

---

## 4. Interview Questions

### Basic
- **Q: Explain the producer-consumer problem and solve it with semaphores.** §2.1 (empty = N, full = 0, mutex = 1).
- **Q: What is the readers-writers problem?** §2.2: concurrent readers, exclusive writers.
- **Q: Why do the dining philosophers deadlock?** All take the left chopstick ⇒ circular wait.

### Intermediate
- **Q: In the bounded buffer, what if you swap `wait(empty)` and `wait(mutex)`?** Deadlock when the buffer is full (§2.1).
- **Q: Who starves in the first readers-writers solution? How do you fix it?** Writers; use writer preference or a fair turnstile.
- **Q: Give 3 solutions to dining philosophers.** Resource ordering, N−1 seats, pick both atomically (§2.3 table).
- **Q [AMZN][QCOM]: Print numbers 1..N alternately using two (or K) threads.** §2.4 code.

### Advanced / killer follow-ups
- **Q: Implement a thread-safe blocking queue in C++.** §2.1 condvar version. Follow-ups: *why two condvars? why `while`? why unlock before notify?* (Unlocking first avoids waking a thread that immediately blocks on the mutex; notifying under the lock is also correct.)
- **Q: Implement a reusable barrier. Why do you need the generation counter?** Without it, a fast thread can pass the next round's barrier before slow threads have woken from this round (§2.5).
- **Q: Transfer between two bank accounts without deadlock?** Lock in address/ID order, or `std::scoped_lock(a.m, b.m)`.
- **Q [HFT]: Your producer-consumer queue between the feed thread and the strategy thread is slow. What do you change?** SPSC lock-free ring buffer, power-of-two size, head/tail on separate cache lines, cached indices, busy-polling consumer, pinned threads on the same NUMA node / sharing an L2 or L3.

---

## 5. Common Traps
- Taking the mutex before waiting on the counting semaphore (deadlock).
- `if` instead of `while` with condition variables.
- `notify_one` when waiters wait on **different** conditions (the wrong thread wakes and the right one sleeps) ⇒ use separate condvars or `notify_all`.
- Forgetting the termination condition in alternation problems (threads wait forever at the end).
- Solving dining philosophers with "try again after random sleep" and calling it deadlock-free (that's livelock-prone).

---

## 6. Revision Summary
| Problem | Key solution |
|---|---|
| Bounded buffer | empty = N, full = 0, mutex = 1; counting `wait` **before** mutex |
| Blocking queue (C++) | mutex + `not_full` + `not_empty` condvars, predicate waits |
| SPSC fast path | lock-free ring buffer (HFT) |
| Readers-writers | `read_count`; first reader locks, last unlocks; reader-pref starves writers |
| Dining philosophers | circular wait → lock ordering / N−1 seats / both-or-none |
| Multi-lock code | global lock order or `std::scoped_lock` |
| Alternation | shared turn variable + condvar (`notify_all`), or baton-passing semaphores |
| Ordering A→B | semaphore(0) / future / latch |
| Barrier | count + generation counter; `std::barrier`, `__syncthreads()` |
