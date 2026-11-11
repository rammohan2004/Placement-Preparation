# 10. Cache Coherence & Memory Consistency ⭐ (HFT favourite)

> **Asked at:** [HFT] [DESHAW] false sharing, cost of atomics, lock-free queues, memory ordering, `std::atomic` memory orders · [APPLE] [QCOM] [NVIDIA] MESI, ARM weak ordering, GPU memory model · [AMZN] concept checks ("why is my multithreaded counter slow / wrong?").
> **Prerequisites:** [06 Cache](06-Cache-Memory.md) (write-back caches, store buffer §2.6.2), [09 OoO](09-ILP-Superscalar-Out-of-Order.md) (loads can run early).
> **Software-role depth:** everything here is asked in SW / systems / HFT interviews **except** the items marked *(HW depth — skim)*. The original compact version is in `archive/10-Coherence-Consistency-original-compact.md`.

---

## 1. Intuition

### 1.1 The setup: every core has its own private cache
```
     Core 0                    Core 1
 ┌──────────────┐          ┌──────────────┐
 │ store buffer │          │ store buffer │   ← stores wait here first (note 06 §2.6.2)
 ├──────────────┤          ├──────────────┤
 │   L1 / L2    │          │   L1 / L2    │   ← PRIVATE: only this core reads/writes it
 └──────┬───────┘          └──────┬───────┘
        └──────────┬──────────────┘
          interconnect (bus / ring / mesh)
                   │
          shared L3  +  main memory
```
Because each core keeps **its own copy** of data, two different problems appear:

| Problem | Question it answers | Name | Analogy |
|---|---|---|---|
| **Stale copies of ONE variable** | If core 1 writes `x`, will core 0 ever stop reading the old `x`? | **Coherence** | Several people have photocopies of the same page; when one edits it, the others must throw away their copies. |
| **Order between DIFFERENT variables** | If core 1 writes `data` then `flag`, can core 0 see the new `flag` but the old `data`? | **Consistency** (memory model) | You post two letters, A then B. Can your friend get B before A? The memory model is the post office's rule book. |

> **One-line difference:** coherence = "everyone agrees on the history of **one address**". Consistency = "what **order** can other cores see my writes to **different addresses**". Coherence is a cache property; consistency is a property of the whole machine (store buffer + OoO + compiler) and is part of the ISA contract.

### 1.2 Why software people must care
- **Correctness:** lock-free code, flags between threads and `std::atomic` memory orders only make sense once you know the memory model.
- **Performance:** coherence moves data in whole **64-byte lines**. Moving a line between cores costs **~50–100 ns** (vs ~1 ns for an L1 hit). False sharing and contended atomics are both "the line keeps moving" problems.

---

# PART A — COHERENCE (one address)

## 2. Deep Dive — Coherence

### 2.1 The problem, step by step
Caches are write-back (note 06): a write updates only the writer's cache, not memory.
```
Start: x = 0 in memory.

Step 1: Core 0 reads x   → Core 0's cache: x = 0
Step 2: Core 1 reads x   → Core 1's cache: x = 0
Step 3: Core 0 writes x=1 → Core 0's cache: x = 1   (memory still 0, Core 1 still 0)
Step 4: Core 1 reads x   → hits in ITS OWN cache → gets 0  ✗ STALE, forever
```
Without a protocol, `while (!stop) {}` on core 1 might **never** see `stop = true` written by core 0. A coherence protocol fixes this **in hardware**, invisibly to software.

### 2.2 The two rules every coherence protocol enforces
1. **Single-Writer / Multiple-Readers (SWMR):** at any moment, a given line is either
   - **writable by exactly ONE core** (nobody else has a copy), or
   - **read-only in any number of cores**.
   Never "one writing while another reads an old copy".
2. **Latest-value rule:** a read returns the value of the most recent write to that address (all cores agree on one order of writes to that address).

Analogy: a library book. Many people can **read photocopies**, but to **edit** the original you must collect and shred everyone's photocopies first.

> Coherence works on **whole cache lines (64 B)**, not on variables. Keep this in mind: it is the root cause of false sharing (§2.6).

### 2.3 How caches find out about each other: snooping vs directory
To shred the other copies, a core must **tell** the others. There are two ways:
```
SNOOPING (broadcast)                     DIRECTORY (look-up table)
Core0  Core1  Core2  Core3               Core0  Core1  Core2  Core3
  │      │      │      │                   │                    ▲
  └──────┴──┬───┴──────┘                   │ 1. "I want to       │ 3. invalidate
     shared bus: everyone                  │    write line L"    │    only the
     hears every request                   ▼                    │    sharers
                                        Directory: "L is held by Core3"
```
| | Snooping | Directory |
|---|---|---|
| How | every miss is **broadcast**; every cache "snoops" (listens) and checks if it has the line | a **home node** keeps a list of which cores hold each line; sends messages **only to those cores** |
| Latency | low (one broadcast) | an extra hop (requester → directory → owner) |
| Scales to many cores? | no (broadcast traffic grows with core count) | yes (servers, multi-socket) |

**Software depth:** know this trade-off in one line. Modern chips mix both (a "snoop filter" in the shared L3 acts as a directory). Directory sizing and the like are *(HW depth — skim)*.

### 2.4 MESI: the state of each line in each cache
Every line in every private cache carries a 2-bit state. The easiest way to understand MESI is to ask, for each state: **"what can I do WITHOUT asking anyone?"**

| State | Meaning | Read without asking? | Write without asking? | Matches memory? |
|---|---|---|---|---|
| **M** – Modified | I have the **only** copy and I **changed** it | ✅ | ✅ | ❌ dirty (memory is stale) |
| **E** – Exclusive | I have the **only** copy, **unchanged** | ✅ | ✅ (silently becomes M) | ✅ clean |
| **S** – Shared | others **may** also have a copy | ✅ | ❌ must invalidate others first | ✅ clean |
| **I** – Invalid | I don't have a valid copy | ❌ miss | ❌ miss | — |

SWMR in MESI terms: **M or E ⇒ nobody else has it**; **S ⇒ nobody can write it**.

#### What my core does (local events)
| I am in | My core... | What I send on the interconnect | I become |
|---|---|---|---|
| I | reads, nobody else has it | **BusRd** ("I want to read L") | **E** |
| I | reads, someone else has it | **BusRd** | **S** |
| I | writes | **BusRdX** = read-for-ownership (RFO): "give me L **and** everyone else invalidate" | **M** |
| S | writes | **BusUpgr** / invalidate: "everyone else drop L" (I already have the data) | **M** |
| E | writes | **nothing** (silent upgrade) | **M** |
| M | reads or writes | nothing (hit) | M |

#### What I hear from other cores (snooped events)
| I am in | Another core... | What I do | I become |
|---|---|---|---|
| M | reads L | send my dirty data (and write it back to memory) | **S** |
| M | wants to write L (RdX) | send my dirty data | **I** |
| E | reads L | (memory or I supply data) | **S** |
| E or S | wants to write L | drop my copy | **I** |

> **"M + remote write ⇒ send my dirty data". Why send it if the other core is going to write anyway?**
> Because coherence works on the **whole 64 B line**, while a write usually changes only a few bytes. The writer still needs the **latest values of the rest of the line**, and in M those exist **only in my cache** (memory is stale).
> ```
> L = [a | b | ...]   memory: a=0, b=0
> Core 0 writes a=5            → Core 0: L in M = [a=5, b=0]   (memory still a=0)
> Core 1 wants to write b=7    → RdX
>   ✗ if Core 0 just drops L:  Core 1 reads memory [a=0,b=0] → writes b → [a=0, b=7]  ⇒ a=5 LOST
>   ✓ Core 0 sends its line:   Core 1 gets [a=5,b=0]        → writes b → [a=5, b=7]  ⇒ both kept
> ```
> That's why it's called **Read**-for-Ownership: get the current bytes + ownership, then merge your write into them.
> - **Why not write back to memory and let the writer read from there?** That would be correct but slow (two trips to memory). A cache-to-cache transfer is faster.
> - **Exception:** if the writer will overwrite the **entire line** (full-line/streaming stores, `rep stosb`/`memset`, ARM `DC ZVA`), the old data really is useless. Hardware then uses an "invalidate-without-data" request (Intel: *ItoM*) and nothing is sent.
> - **Why doesn't S (or E) need to send data on a remote write?** S/E copies are **clean**: memory/L3 already has the same bytes, so dropping them loses nothing. (A core may still supply clean data for speed, but it isn't needed for correctness.)

#### All transitions on one screen
```
 MY core does something (I may need to ASK)     ANOTHER core does something (I must OBEY)
 ──────────────────────────────────────────     ─────────────────────────────────────────
 I ── read, nobody has it  (BusRd) ──► E        M ── it reads  (I send dirty data) ──► S
 I ── read, others have it (BusRd) ──► S        M ── it writes (I send dirty data) ──► I
 I ── write           (BusRdX/RFO) ──► M        E ── it reads ───────────────────────► S
 S ── write              (BusUpgr) ──► M        E ── it writes ──────────────────────► I
 E ── write             (SILENT!)  ──► M        S ── it writes ──────────────────────► I
 M/E/S ── read: hit, nothing sent               I ── anything: I have nothing, ignore
```
Pattern to remember: **remote write ⇒ I always go to I**. **Remote read ⇒ I drop to S** (and supply the data if I was M).

#### "L1 is private, so how does a write ever reach the other cores?"
**A write hit that stays inside the core is always safe**, because MESI allows a silent write only when no other copy exists.
| My L1 state | Read | Write |
|---|---|---|
| M | hit, nothing sent | hit, **nothing sent** (only owner) |
| E | hit, nothing sent | hit, **nothing sent** (E→M silently) |
| S | hit, nothing sent | ❌ **"hit without permission"** ⇒ send Upgrade, wait (store waits in the store buffer) |
| I | miss ⇒ request | miss ⇒ RFO |
"Private" means only this core **reads/writes** it. Every L1/L2 still has a **coherence controller** that sends requests out and **receives snoops** (invalidate/read) from the interconnect.
```
  Core 0 [SB | L1+ctrl | L2]          Core 1 [SB | L1+ctrl | L2]   ← both receive snoops
 ════════════╪══════════ ring / mesh ═══════════╪═══════════
             ▼                                  ▼
  L3 slices + SNOOP FILTER / DIRECTORY: line X → sharers {0,1}, state   ← knows WHO may hold X
```
(Textbook MESI draws a shared bus that every L1 watches. Modern chips track sharers at the L3 and snoop only those cores.)

**Core 0 writes X while both cores hold it in S:**
1. Core 0's L1: tag match, state S ⇒ no write permission ⇒ the store waits in the store buffer.
2. Core 0's L1/L2 sends "Upgrade X" over the ring to X's home L3 slice.
3. L3 snoop filter: sharers = {0, 1}.
4. L3 sends "invalidate X" to core 1 (its L2 forwards it to its L1).
5. Core 1: S→I, sends an ack.
6. L3 records "owner = core 0" and replies "granted".
7. Core 0: S→M; the store buffer drains into L1. Later writes are silent M hits.

**Why silent M/E hits don't confuse the L3:** when core 0 got E/M, the L3 **already recorded it as exclusive owner**. The L3 doesn't need to hear about every write, only **whom to ask** next time. If core 1 later reads X: miss → L3 → "owner = core 0" → snoop core 0 → core 0 sends the dirty data, M→S. The L3's own copy may be stale meanwhile; it knows to fetch from the owner. (E vs M: the L3 treats both as "core 0 owns it"; the snoop reply says whether the line is dirty.)

#### Why does E exist? (classic question)
Most data is **private** to one thread (its stack, local arrays). Typical pattern: **read, then write**.
- **With E (MESI):** read → miss, nobody else has it → **E**. Write → E→M silently. **1** bus transaction.
- **Without E (MSI):** read → **S** (the cache can't know it's alone). Write → must broadcast an invalidate. **2** bus transactions, for data nobody shares.
So E saves an invalidate broadcast for every "read-then-write" of private data.

#### Variants (know the name and one line)
- **MOESI (AMD):** adds **O**wned = "dirty but shared". On a remote read, M → **O** instead of M → S. The owner keeps supplying data and **skips the write-back to memory**.
- **MESIF (Intel):** adds **F**orward = one chosen sharer answers read requests, so several S copies don't all reply at once.
- All modern CPUs use **write-invalidate** (invalidate others, then write). **Write-update** (broadcast the new value to all copies) causes too much traffic and is rare.

### 2.5 What a coherence miss costs (numbers to quote)
| Where the line comes from | Typical latency |
|---|---|
| Own L1 hit | ~1 ns (4–5 cycles) |
| Own L2 hit | ~3–4 ns |
| Shared L3 hit (clean) | ~10–20 ns |
| **Another core's private cache, same socket** (e.g. the line is M there) | **~40–100 ns** |
| Another socket (NUMA) | ~100–200 ns |

Every coherence-related slowdown in software is "a line I need is sitting in M/E in **another** core's cache, so I pay row 4 instead of row 1".

### 2.6 False sharing [HFT][DESHAW][NVIDIA][AMZN]
**Definition:** two threads write **different variables** that happen to sit on the **same 64-byte line**. Logically nothing is shared, but coherence works per line, so the line bounces between the cores.

```cpp
struct Counters {
    std::atomic<long> a;   // bytes 0..7   ┐ same 64 B line
    std::atomic<long> b;   // bytes 8..15  ┘
};
// Thread A (core 0): for(...) c.a++;      Thread B (core 1): for(...) c.b++;
```
Step by step (line L holds both `a` and `b`):
| Time | Event | Core 0's copy of L | Core 1's copy of L | Cost |
|---|---|---|---|---|
| t1 | core 0 writes `a` → RFO | **M** | I | transfer |
| t2 | core 1 writes `b` → L is I locally → RFO; core 0 sends L | I | **M** | ~70 ns |
| t3 | core 0 writes `a` → miss again → RFO | **M** | I | ~70 ns |
| t4 | core 1 writes `b` → miss again | I | **M** | ~70 ns |
| … | the line "ping-pongs" on every write | | | |
Each increment should take ~1 ns (L1 hit) but takes ~70 ns ⇒ **10–100× slowdown**, with no bug visible in the code.

**Fix: give each variable its own line.**
```cpp
struct alignas(64) PaddedCounter { std::atomic<long> v; };   // sizeof == 64
PaddedCounter counters[NTHREADS];                            // one line per thread
// Portable C++17: alignas(std::hardware_destructive_interference_size)
```
- Many HFT codebases pad to **128 B**: Intel's adjacent-line prefetcher pulls lines in **pairs**, and Apple M-series uses **128-byte** lines.
- Other fixes: per-thread local variable, combined once at the end (best); batch updates.
- **Detect:** `perf c2c` (looks for HITM = "hit a line Modified in another core"), Intel VTune.

> **True sharing vs false sharing:** if both threads really update the **same** variable, that's true sharing: padding can't help, so redesign (per-thread counters plus a final sum).

### 2.7 Atomics and locks — what they really do in the cache

#### 2.7.1 Coherence does NOT make `count++` safe
`count++` is **three** instructions: load, add, store. Coherence keeps each **single** load or store correct, but other cores can get in **between** them.
```
count = 0 (one shared int, two threads each do count++ once)

  Core 0                     Core 1
  load  r = count  (0)
                             load  r = count  (0)
  add   r = r+1    (1)
                             add   r = r+1    (1)
  store count = r  (1)
                             store count = r  (1)    ← final 1, expected 2: LOST UPDATE
```
No coherence rule was broken: each load saw the latest value at that moment. The problem is that load…store was **not one indivisible step**. (In C++ this is also a data race = undefined behaviour.)

#### 2.7.2 How an atomic RMW (read-modify-write) works
`count.fetch_add(1)` → x86 `lock xadd`, ARM `LDADD` (or an `LDXR/STXR` loop):
```
1. Core gets line L in M state (RFO: everyone else invalidated).
2. Core LOCKS the line: if another core asks for L, the request is delayed.
3. Load, add, store, all on its own copy.
4. Unlock: now other cores' requests are served.
⇒ Nobody can get in between the load and the store. No lost update.
```
**Cost:**
- **Uncontended** (line already in my cache): ~10–25 cycles on x86, more if stores are pending. On x86 a `lock` instruction also acts as a **full fence**: before it runs, all of this core's earlier stores still waiting in the store buffer must be written into L1 (made visible to all cores), and later loads wait for it. (What a fence is and why `lock` implies one: §3.5.1–3.5.5.)
- **Contended** (many cores updating the same atomic): every operation first has to pull the line from another core (§2.5), so **~100s of cycles per op**, all serialised. Throughput **drops** as you add threads.

#### 2.7.3 Spinlocks: why test-and-test-and-set is better
```cpp
// (a) Naive test-and-set
while (lock.exchange(true)) {}            // exchange = an atomic WRITE, every iteration

// (b) Test-and-test-and-set (TTAS)
while (true) {
    while (lock.load(std::memory_order_relaxed)) _mm_pause();  // spin on a READ
    if (!lock.exchange(true, std::memory_order_acquire)) break; // try only when it looks free
}
```
Explained with MESI (lock held by core 0; cores 1, 2, 3 waiting):
| | (a) test-and-set | (b) TTAS |
|---|---|---|
| What each waiter does per spin | atomic write ⇒ **RFO** ⇒ steal the line in M | plain read ⇒ line stays in **S** in its own cache |
| Interconnect traffic while waiting | line ping-pongs among cores 1, 2, 3 nonstop | **none**: each waiter spins in its own L1 |
| Effect on the lock holder | its unlock store must fight for the line ⇒ slower unlock | unlock invalidates the S copies once; waiters miss, see "free", one wins the exchange |
- `pause` (x86) / `yield` (ARM): tells the core it's in a spin loop (saves power, helps the hyper-thread sibling).
- Fairer / more scalable locks: **ticket lock** (FIFO order), **MCS/CLH** queue locks (each waiter spins on **its own** line).

#### 2.7.4 CAS, ABA, LL/SC (one-liners interviewers check)
- **CAS** (compare-and-swap, `compare_exchange_strong/weak`): "if the value is still A, set it to B; else tell me the current value". The building block of lock-free structures.
- **ABA problem:** thread 1 reads A, gets delayed; others change A→B→A; thread 1's CAS succeeds even though the structure changed (e.g. a freed and reused node). Fix: tagged pointers / version counter, hazard pointers, epoch-based reclamation.
- **LL/SC** (ARM `LDXR/STXR`, RISC-V `LR/SC`): the store-conditional fails if **anyone wrote the line** since the load-linked ⇒ no ABA by construction, but it can fail spuriously (that's why `compare_exchange_weak` exists: use it in loops).

---

# PART B — CONSISTENCY (order across addresses)

## 3. Deep Dive — Memory Consistency

### 3.1 Why coherence is not enough
```cpp
// Producer (core 0)            // Consumer (core 1)
data = 42;                      while (flag == 0) {}
flag = 1;                       print(data);        // 42? or old 0?
```
`data` and `flag` are **different addresses** (different lines). Coherence promises that core 1 **eventually** sees `data = 42` and **eventually** sees `flag = 1`. It promises **nothing about which it sees first**. If core 1 sees the new `flag` but the old `data`, it prints 0. The rule that decides this is the **memory consistency model**.

### 3.2 Sequential Consistency (SC): the "intuitive" model
**Definition (Lamport):** the result is as if all cores' operations were put into **one single sequence** (an interleaving), and each core's operations appear in that sequence **in its program order**.

Think of one global "turn-taking" order: at each step one core does its next instruction.

**Test it: the store-buffer (Dekker) litmus test**
```
Initially x = y = 0
   Core 0          Core 1
A: x = 1        C: y = 1
B: r1 = y       D: r2 = x
Question: can we end with r1 == 0 AND r2 == 0 ?
```
Under SC, list **every** interleaving that keeps A before B and C before D (4!/(2!·2!) = **6** of them):
| # | Order | r1 (=y at B) | r2 (=x at D) |
|---|---|---|---|
| 1 | A B C D | 0 | 1 |
| 2 | A C B D | 1 | 1 |
| 3 | A C D B | 1 | 1 |
| 4 | C A B D | 1 | 1 |
| 5 | C A D B | 1 | 1 |
| 6 | C D A B | 1 | 0 |
**(0, 0) never appears.** Reason: whichever store happens first in the sequence comes before the other core's load, so at least one load sees a 1.

SC is easy to reason about, but it **forbids the store buffer and out-of-order loads**, and real CPUs won't give those up.

### 3.3 How the store buffer breaks SC (step by step)
Recall note 06 §2.6.2: a store goes into the core's **store buffer** first and reaches L1 (becomes visible to others) only later, once the line is in M. Meanwhile **later loads don't wait**.
```
Time  Core 0                              Core 1
 t1   A: x = 1 → into SB0 (not visible)    C: y = 1 → into SB1 (not visible)
 t2   B: r1 = y → SB0 has no y →           D: r2 = x → SB1 has no x →
         read cache: y = 0                    read cache: x = 0
 t3   SB0 drains: x = 1 visible            SB1 drains: y = 1 visible   (too late)
Result: r1 = 0, r2 = 0   ← impossible under SC, REAL on x86 and ARM
```
From outside, it looks as if each core did its **load before its store**: a **Store → Load reordering**.

> **"But doesn't the load read from the store buffer?"** Only if it's the **same address** (store-to-load forwarding: a core always sees its own stores). Store→Load reordering is about **different** addresses (`x = 1; r1 = y;`): the store buffer has nothing for `y`, so the load reads `y` from the **cache**.
>
> **Where exactly is the reordering?** A memory op "happens" (for other cores) when it takes effect in the shared memory system: a **store** when it is written into L1 (not when it enters the store buffer), a **load** when it reads its value.
> ```
> t1  x = 1  executes → into store buffer        (not yet happened for others)
> t2  r1 = y → no y in SB → reads cache           ← load HAPPENS
> t3  SB drains, x = 1 written into L1            ← store HAPPENS
> Program order: store, load.   Order others observe: load, store  ⇒ reordered
> ```
> Instructions **executed** in order, but **took effect** in the opposite order. "Memory reordering" always means the order **others can observe**.
>
> **Caused by OoO? No: by the store buffer.** Even an **in-order** core has a store buffer (otherwise every store miss would stall ~100 cycles), so Store→Load reordering, and the Dekker (0,0) result, happens on in-order cores too.
> | Reordering | Hardware cause | In-order core too? |
> |---|---|---|
> | Store → Load | store buffer (loads don't wait for older stores to drain) | ✅ yes |
> | Store → Store | store buffer draining out of order (ARM merging; a hit draining before a miss) | ✅ possible |
> | Load → Load | OoO execution; also non-blocking caches ("hit under miss") | mostly OoO |
> | Load → Store | OoO / speculation | mostly OoO |
> *Why not remove the store buffer to get SC?* Every store would wait for the line in M (100+ cycles on a miss). Far too slow, so CPUs keep the store buffer and provide fences.

**Fix: a full fence between the store and the load**
```
Core 0: x = 1;  mfence;  r1 = y;      // ARM: dmb ish   C++: seq_cst atomics
```
`mfence` = "**wait until my store buffer has drained** (my earlier stores are visible) before running later loads".
Why that's enough: now each core's store is visible **before** its own load. Take whichever load runs later in time; by then **both** stores are visible, so that load reads 1. ⇒ (0, 0) is impossible.

### 3.4 The memory models of real CPUs
Name the 4 possible reorderings as "**earlier op → later op**" (in program order) that may **appear swapped** to other cores:

| May appear reordered? | SC | **x86-64 (TSO)** | **ARMv8 / RISC-V / POWER (weak)** | Hardware reason |
|---|---|---|---|---|
| Store → Load (a later load passes an earlier store) | no | **YES** | yes | store buffer (§3.3) |
| Store → Store | no | no | **yes** | ARM's store buffer may drain out of order (e.g. 2nd store hits, 1st misses) |
| Load → Load | no | no | **yes** | OoO core runs a later load first (e.g. 1st load misses) |
| Load → Store | no | no | **yes** | OoO / speculation |

- **x86 = TSO (Total Store Order)**, a "strong" model: the store buffer is a **FIFO** (stores become visible in program order), and loads appear in order. x86 OoO cores *do* run loads early internally, but if the line is invalidated before the load retires, the core **squashes and replays** it, so nobody can tell. **Only Store→Load reordering is visible.**
- **ARM / RISC-V (RVWMO) = weak:** almost anything can appear reordered unless you add barriers or acquire/release instructions. This gives simpler, lower-power hardware, and puts the burden on software.
- **[APPLE] trivia:** Apple M-series cores have a **hardware TSO mode**, switched on for Rosetta 2–translated x86 apps, so x86 multithreaded code that relies on TSO still works on ARM.

#### TSO in one table: what the OoO core does inside vs what others can see
The x86 OoO core still reorders **internally**; TSO only promises what other cores can **observe**.
| Program order | Visible to others on x86? | What happens inside | How it's hidden |
|---|---|---|---|
| load → later load | ❌ | later load runs early when the earlier one misses | snoop check: line invalidated before commit ⇒ squash and re-run (next subsection) |
| load → later store | ❌ | store's address/data computed early | store becomes visible only **after commit**; commit is in order ⇒ the earlier load already has its value |
| store → later store | ❌ | both wait in the store buffer | **FIFO** store buffer drains in program order |
| store → later load (**different** address) | ✅ **allowed** | load finds nothing for its address in the store buffer ⇒ reads the **cache** while the store still waits in the store buffer | **not hidden** ⇒ Dekker needs `mfence` |
Same-address exception: a load to an address with a pending store gets the value **forwarded from the store buffer**, so a core sees its own stores before other cores do.
**Interview line:** "x86 TSO: the only visible reordering is a later load passing an earlier store to a different address (store buffer). ARM/RISC-V can show all four."

> **Is TSO just "the store buffer drains in order"?** That's only **half**. TSO = (1) **stores** become visible in program order (FIFO store buffer) **and** (2) **loads** appear in program order (early loads squashed if their line changes). ARM/RISC-V guarantee **neither**.
> The ISA doesn't mandate a store buffer; it specifies which orders others may observe. A FIFO store buffer is how x86 hardware keeps promise (1).
> **Why ARM lets stores drain out of order:** if the oldest store misses (100 cycles) and a younger one hits, the younger one can drain now, with no head-of-line blocking, plus stores to the same line can merge. Simpler, faster, lower-power hardware; software adds `stlr`/`dmb` where order matters. (x86 eases its blocking by prefetching ownership for stores waiting in the buffer.)
> **"Not guaranteed" ≠ "always happens":** a given ARM chip may usually drain in order, so buggy code passes tests and fails on another chip. Code to the ISA, not to one chip.
> **"Total" in TSO:** all cores agree on one single order of all stores.

#### What "the OoO core runs loads early" means (source of Load → Load reordering)
An OoO core runs any instruction whose inputs are ready, even if an **older** instruction is stuck (note 09); it only **commits** in order. So a **later load can read memory before an earlier load does**.
```
I1: r1 = flag     ← cache MISS (~100 cycles)
I2: r2 = data     ← cache HIT  (~4 cycles)
In-order core: I2 waits ~100 cycles behind I1.
OoO core:      I2 doesn't depend on I1 ⇒ reads data at cycle ~2, ~95 cycles BEFORE flag is read.
```
**Single thread:** harmless (nobody changed `data` in between) and much faster.
**Another core writes in between** (producer runs `data = 42; flag = 1;`):
```
Cycle   Core 1 (consumer, OoO)                      Core 0 (producer)
  1     I1: r1 = flag → MISS, request sent
  2     I2: r2 = data → HIT → reads OLD data = 0
 50                                                 data = 42 ; flag = 1
100     I1 completes → r1 = flag = 1 (NEW)
⇒ r1 = 1, r2 = 0: looks as if the two loads swapped (Load → Load reordering)
```
| | x86 (TSO) | ARM / RISC-V (weak) |
|---|---|---|
| Runs I2 early? | yes | yes |
| `data`'s line invalidated before I2 commits? | core **detects** the snoop, **squashes and re-executes** I2 ⇒ gets 42 | keeps the early value (allowed) |
| Visible to software? | **no** | **yes** ⇒ need **acquire** on the `flag` load (`ldar`) |
So running loads early is a hardware speed trick in **every** OoO core; the memory model decides whether software may **see** it. (The store buffer is the other source: a later load runs before an earlier **store** is visible = Store → Load, §3.3.)

#### The message-passing example on x86 vs ARM
```
P0: data = 1;  flag = 1;            P1: r1 = flag;  r2 = data;
Can P1 end with r1 = 1, r2 = 0 ?
```
- **x86: NO.** Stores leave P0 in order (FIFO store buffer) and P1's loads appear in order. If P1 saw `flag = 1`, then `data = 1` was already visible.
- **ARM: YES, in two independent ways:**
  1. **Producer side (Store→Store):** the `data` line misses and the `flag` line hits, so `flag = 1` becomes visible first.
  2. **Consumer side (Load→Load):** P1 runs `r2 = data` early (while `flag` is still missing in the cache), gets the old 0, then reads `flag = 1`.
  ⇒ You must fix **both** sides: a **release** store for `flag` in P0 and an **acquire** load of `flag` in P1.

### 3.5 Fences (memory barriers) and acquire / release

#### 3.5.1 The problem fences solve
To go fast, the CPU **reorders memory operations**: stores wait in the store buffer (§3.3), and the OoO core runs a later load before an earlier one that is stuck on a miss (§3.4, "runs loads early").
- **Your own thread never notices.** The core checks its own store buffer (store-to-load forwarding) and commits in order, so a single thread always sees its own program order.
- **Other cores can notice.** They may see your memory ops land in a different order.
```
What Core 0 wrote:     x = 1;   r1 = y;
What Core 1 can see:   r1 = y happened first; x = 1 became visible later
                       (x = 1 was still sitting in Core 0's store buffer)
```

#### 3.5.2 What a fence is
A **fence / memory barrier** is an **ISA instruction** that means: *"every memory op **before** me must be complete and visible to all cores before **any** memory op **after** me is performed."* Analogy: "don't send parcel 2 until parcel 1 has been signed for."

What `mfence` does in hardware:
```
x = 1;     → enters the store buffer
mfence;    → core stops performing later loads/stores; WAITS until the store buffer is empty
             (x = 1 written into L1, line in M, other copies invalidated ⇒ visible to all)
r1 = y;    → only now executes
```
**Cost:** the time to drain the store buffer. That's tens of cycles, or **hundreds** if a pending store misses in the cache. A **full** fence stops all four reorderings in the §3.4 table.

#### 3.5.3 When you need one
Only when **two observers communicate through memory** (thread ↔ thread, CPU ↔ device/DMA). **A single thread on normal memory never needs a fence.**
- **Dekker / Peterson** (§3.3): store-then-load-another-variable needs a full fence **even on x86**.
- **Mutex:** `lock()` must stop critical-section ops moving **above** it, and `unlock()` must stop them moving **below** it. Otherwise, on ARM, a write made inside the critical section could become visible after the unlock, and the next thread would read stale data. `std::mutex` puts the right barriers in for you.
- **Message passing** (data → flag): on x86 no fence is needed (TSO), only a compiler barrier. On ARM it needs release/acquire (§3.5.7).

#### 3.5.4 Fences are part of the ISA
| Purpose | x86 | ARMv8 | RISC-V |
|---|---|---|---|
| Full fence (nothing crosses) | `mfence`; **any `lock`-prefixed instruction**; `xchg` with memory (implicitly locked) | `dmb ish` | `fence rw,rw` |
| Keep stores ordered | `sfence` (only needed for non-temporal/streaming stores; normal x86 stores are already ordered) | `dmb ishst` | `fence w,w` |
| Keep loads ordered | `lfence` (mostly used to serialise `rdtsc` timing / as a Spectre barrier) | `dmb ishld` | `fence r,rw` |
| One-way (acquire / release) | not needed (TSO already gives it) | `ldar`/`ldapr`, `stlr` | `.aq` / `.rl` bits on AMOs, LR/SC |
| Embedded extras [TI][QCOM] | — | `dsb` = wait until earlier ops **complete** (MMIO, DMA, cache maintenance); `isb` = flush the pipeline (after changing system registers / enabling the MMU) | `fence.i` (instruction-fetch sync) |

- **Compiler barrier** `asm volatile("" ::: "memory")`: **not** an ISA instruction. It stops only the **compiler** from reordering, not the CPU.
- **C++ level:** `std::atomic_thread_fence(order)`, though usually you just use atomics with memory orders and the compiler emits the right instruction. Linux kernel: `smp_mb()`, `smp_rmb()`, `smp_wmb()`.

#### 3.5.5 Why a `lock` instruction is a full fence on x86
1. A locked RMW (`lock xadd`, `lock cmpxchg`, `xchg`) must be **atomic and visible to all cores at once**, so its store **cannot wait** in the store buffer.
2. The x86 store buffer is **FIFO** (stores become visible in program order). For the locked store to be visible now, **every earlier store must become visible first** ⇒ the store buffer must drain.
3. The ISA also says later loads can't pass a locked instruction.
⇒ A `lock`ed instruction = **atomic op + full fence**. That's why even an uncontended atomic costs ~20+ cycles, and why compilers implement a seq_cst store as `xchg` (one instruction gives the store + fence; often cheaper than `mov` + `mfence`).
On **ARM**, atomics are **not** automatically fences. The variant picks the ordering: `ldadd` (relaxed), `ldadda` (acquire), `ldaddl` (release), `ldaddal` (both).

#### 3.5.6 How much of this do you need?
| Know it | Skim / skip |
|---|---|
| What a fence is; the store buffer / OoO as the reason | `sfence`/`lfence` semantics with write-combining memory |
| x86 needs a fence only for store→load (Dekker, seq_cst store); ARM needs more | POWER `lwsync` / `hwsync`, multi-copy atomicity |
| `std::atomic` and `std::mutex` emit fences for you; you rarely write `mfence` by hand | exact cycle counts of each fence |
| A `lock` op is a full fence on x86; the cost of draining the store buffer | |
| [TI][QCOM] `dmb`/`dsb`/`isb` one-liners · [HFT] `lfence; rdtsc` / `rdtscp` for timing | |

#### 3.5.7 Acquire and release: one-way fences
A **full fence** blocks movement **in both directions** and is expensive. **Acquire and release block only one direction**: cheaper, and exactly what lock / unlock and publish / subscribe need.
```
        ordinary ops above ...          ↓ these MAY sink into the region
 ════ ACQUIRE (lock / load-acquire) ════   nothing BELOW may move ABOVE this line
        critical section / reads of published data
 ════ RELEASE (unlock / store-release) ═   nothing ABOVE may move BELOW this line
        ordinary ops below ...          ↑ these MAY rise into the region
```
- **Release store:** "everything I did **before** this store is visible to anyone who sees this store". (Earlier ops can't sink below it.)
- **Acquire load:** "nothing I do **after** this load runs before it". (Later ops can't rise above it.)
- **The pairing (synchronizes-with):** if an **acquire load reads the value written by a release store**, then everything before the release (in the producer) is visible to everything after the acquire (in the consumer).

Fixed message passing:
```cpp
std::atomic<bool> ready{false};
int data;
// Producer                                   // Consumer
data = 42;                                    while (!ready.load(std::memory_order_acquire)) {}
ready.store(true, std::memory_order_release); assert(data == 42);   // guaranteed
```
Release = **publish** (data is written before the flag goes up). Acquire = **subscribe** (data is read only after the flag is seen).

> **With a mutex, can the compiler/CPU reorder instructions INSIDE the critical section? Yes, and it's safe.**
> `lock()` = acquire, `unlock()` = release, so:
> | Movement | Allowed? | Why |
> |---|---|---|
> | reordering **inside** the CS | ✅ | nobody can observe it |
> | inside op moves **out** (above lock / below unlock) | ❌ | would expose shared data without the lock |
> | outside op moves **in** | ✅ | more protection never hurts ("roach motel") |
> Why it's invisible: another thread can't look at the shared data **during** the CS (it's blocked in `lock`). When it gets the lock, the release in `unlock` has made **all** CS writes visible, and its own acquire stops its reads from running early, so it sees only the **final** state. The owning thread sees its own ops in program order anyway (as-if rule).
> Implementation: x86 `lock()` = `lock cmpxchg` (full fence), `unlock()` = release store / `xchg`; ARM `casa` / `ldaxr`+`stxr` (acquire) and `stlr` (release); the compiler won't move memory ops out across either.
> **It breaks only if someone touches the shared data WITHOUT the lock:** `T1: lock; a=1; b=1; unlock;` `T2 (no lock): r1=b; r2=a;` ⇒ `r1=1, r2=0` is possible on ARM (and it's a data race / UB in C++ everywhere). Rule: **every** access to shared data, reads included, holds the same mutex.
> **Classic example: double-checked locking** `if (!p) { lock; if (!p) p = new T; unlock; }`. The outer read of `p` is unlocked; inside the CS the store to `p` may become visible before the object's fields ⇒ another thread uses a half-built object. Fix: `std::atomic<T*>` (acquire/release), `std::call_once`, or a function-local `static` (thread-safe since C++11).

### 3.6 C++ `std::atomic` memory orders, and what they compile to
| `memory_order` | Guarantee (plain words) | x86-64 | ARMv8 |
|---|---|---|---|
| `relaxed` | the op itself is atomic (no torn values, no lost RMW updates); **no ordering** of other memory ops. Use for counters/statistics. | plain `mov` (RMW: `lock add`) | plain `ldr`/`str` (RMW: `ldadd`) |
| `acquire` (loads) | later ops stay after it | plain `mov` (TSO already gives this) | `ldar` / `ldapr` |
| `release` (stores) | earlier ops stay before it | plain `mov` (TSO already gives this) | `stlr` |
| `acq_rel` (RMW) | both | `lock xadd`/`lock cmpxchg` | `ldaddal`, `casal` |
| `seq_cst` (default!) | acquire/release **plus** one global order of all seq_cst ops (forbids the Dekker (0,0)) | load: `mov`; **store: `xchg` (or `mov`+`mfence`) — expensive** | `ldar` / `stlr` |

Key takeaways:
- On **x86**, acquire/release are **free** (plain moves); only **seq_cst stores** and RMWs cost a fence. On **ARM**, every ordered access uses a special instruction.
- `std::atomic` also stops the **compiler** from reordering or caching the value (next section). Even on x86, you need it for that reason.
- The default `x.store(v)` is **seq_cst**. In hot paths HFT code uses explicit `release`/`acquire`.

### 3.7 Two layers of reordering, and the C/C++ tools that stop them
Everything in §3.2–3.5 was **hardware** reordering. The **compiler** is a second, independent source.

#### 3.7.1 The two layers
```
 your C++ source
      │   LAYER 1: COMPILER (build time) may reorder, merge, delete, keep values in registers
      ▼
 machine instructions
      │   LAYER 2: CPU (run time) may reorder visibility (store buffer, OoO loads)
      ▼
 the order other cores actually observe
```
| | Compiler reordering | Hardware reordering |
|---|---|---|
| Who | gcc / clang / MSVC at **build time** | the CPU at **run time** |
| Why it's allowed | **as-if rule**: only **single-thread** behaviour must be preserved; C++ says a data race is UB, so the compiler assumes no other thread touches plain variables | store buffer, OoO; the ISA memory model permits it |
| Depends on | compiler, optimisation level | ISA (x86 TSO vs ARM weak), chip |
| Happens on x86? | **yes, fully** | only store→load |
| Stopped by | compiler barrier (**no instruction**) | ISA fence instruction |
The two are **independent**: a compiler barrier doesn't stop the CPU, and an `mfence` in asm without a `"memory"` clobber doesn't stop the compiler. Correct code needs both stopped. `std::atomic` does both from one line.

#### 3.7.2 What the compiler actually does (3 classic bugs)
```cpp
// (1) Reordering independent stores
int data; bool ready;                      // plain variables
data = 42; ready = true;                   // -O2 may emit the store to ready FIRST
                                           // → bug even on x86, whose hardware keeps store order

// (2) Keeping a value in a register (hoisting the load out of the loop)
bool stop = false;
while (!stop) { work(); }                  // if work() doesn't touch stop → becomes
                                           // if (!stop) for(;;) work();   (never exits)

// (3) Merging / deleting stores
progress = 1; progress = 2; progress = 3;  // only "progress = 3" survives
```
All legal for one thread; bugs only when another thread / signal handler / ISR / device is watching.

#### 3.7.3 Compiler-only constructs: software only, NOT in the ISA, emit no instruction
| Construct | Origin | Effect |
|---|---|---|
| `asm volatile("" ::: "memory")` | GCC/Clang (Linux `barrier()`) | no memory access may move across it; register-cached values must be re-read |
| `std::atomic_signal_fence(order)` | C++11 `<atomic>` / C11 | portable compiler-only fence; meant for a thread and its own signal handler |
| `volatile` | C/C++ | every access really happens (not merged/deleted/cached); volatile↔volatile order kept; **non-volatile accesses may still move across** |
| `READ_ONCE()` / `WRITE_ONCE()` | Linux kernel | volatile access to one variable |
| `_ReadWriteBarrier()` | MSVC (deprecated) | compiler barrier |
**Enough on its own only when the hardware can't reorder what matters**, e.g. a **single-core MCU** where an ISR sets a flag (`volatile bool rx_ready;` on Cortex-M) [TI][QCOM]. With multiple cores it's not enough.

`volatile` vs `std::atomic` in detail:
| | `volatile` (C/C++) | `std::atomic` |
|---|---|---|
| Re-read from memory every access (no register caching) | ✅ | ✅ |
| Atomic RMW (`++` without lost updates) | ❌ | ✅ |
| Orders **other** memory ops (data before flag) | ❌ | ✅ (acquire/release/seq_cst) |
| Emits hardware barriers on ARM | ❌ | ✅ |
| Correct use | **MMIO device registers**, signal handlers [TI][QCOM] embedded | **thread synchronisation** |
(Java's `volatile` is different: it has acquire/release semantics. Interviewers like this contrast.)

#### 3.7.4 Constructs that stop BOTH compiler and hardware (from easiest to lowest level)
These are also **not ISA**: they are part of the language/library (the C11/C++11 memory model). The compiler stops its own reordering **and** emits the ISA fence the target needs (sometimes none).
| # | Construct | Example | Notes |
|---|---|---|---|
| 1 | **Mutex** | `std::lock_guard<std::mutex> g(m);` | lock = acquire, unlock = release. Also thread create/join, `future`, `condition_variable`. **Default choice** |
| 2 | **`std::atomic` + `memory_order`** | `ready.store(true, std::memory_order_release);` | main lock-free tool (§3.6) |
| 3 | **`std::atomic_thread_fence(order)`** | `std::atomic_thread_fence(std::memory_order_acquire);` | standalone fence, pairs with relaxed atomics (§3.7.6) |
| 4 | C11 `<stdatomic.h>` | `atomic_store_explicit(&x, 1, memory_order_release);` `atomic_thread_fence(memory_order_seq_cst);` | the same model for C |
| 5 | GCC/Clang builtins | `__atomic_load_n(&x, __ATOMIC_ACQUIRE)`, `__atomic_thread_fence(__ATOMIC_SEQ_CST)`, legacy `__sync_synchronize()` (full) | what `<atomic>` is built on |
| 6 | CPU intrinsics (non-portable) | x86 `<immintrin.h>`: `_mm_mfence()`, `_mm_sfence()`, `_mm_lfence()` · ARM ACLE `<arm_acle.h>`: `__dmb(0xB)`, `__dsb(0xF)`, `__isb(0xF)` · **CMSIS** (Cortex-M): `__DMB()`, `__DSB()`, `__ISB()` · MSVC `MemoryBarrier()` | embedded / driver code |
| 7 | Inline asm | `asm volatile("mfence" ::: "memory");` · `asm volatile("dmb ish" ::: "memory");` | the **`"memory"` clobber** is what makes it a compiler barrier too |
| 8 | Linux kernel | `smp_mb()/smp_rmb()/smp_wmb()` (CPU↔CPU), `mb()/rmb()/wmb()` (also devices), `smp_load_acquire()`, `smp_store_release()`, `dma_wmb()` | know the names |

**What `std::atomic_thread_fence` compiles to:**
| order | x86 | ARMv8 | RISC-V |
|---|---|---|---|
| `acquire` | *nothing* (compiler barrier only) | `dmb ishld` | `fence r,rw` |
| `release` | *nothing* (compiler barrier only) | `dmb ish` | `fence rw,w` |
| `acq_rel` | *nothing* | `dmb ish` | `fence.tso` |
| `seq_cst` | `mfence` | `dmb ish` | `fence rw,rw` |
Key insight: on x86, TSO already gives acquire and release, so the compiler emits **no instruction** and only stops **itself**. The same C++ line becomes a real `dmb` on ARM.

#### 3.7.5 Master table: what stops what
| Construct | Stops compiler? | Stops hardware? | Instruction emitted |
|---|---|---|---|
| `asm volatile("" ::: "memory")` | ✅ | ❌ | none |
| `std::atomic_signal_fence` | ✅ | ❌ | none |
| `volatile` | partly (volatile↔volatile only) | ❌ | plain load/store |
| `asm volatile("mfence")` **without** `"memory"` | ❌ ⚠️ | ✅ | `mfence` |
| `asm volatile("mfence" ::: "memory")`, `_mm_mfence()`, `__DMB()` | ✅ | ✅ | the fence |
| `std::atomic` op with `memory_order` | ✅ | ✅ | depends on ISA (nothing / `ldar` / `stlr` / `xchg` …) |
| `std::atomic_thread_fence` | ✅ | ✅ | depends on ISA (table above) |
| `std::mutex` lock/unlock | ✅ | ✅ | inside the lock implementation |

#### 3.7.6 Fence + relaxed atomics pattern
```cpp
// Producer                                         // Consumer
data = 42;                                          while (!ready.load(std::memory_order_relaxed)) {}
std::atomic_thread_fence(std::memory_order_release); std::atomic_thread_fence(std::memory_order_acquire);
ready.store(true, std::memory_order_relaxed);       assert(data == 42);   // guaranteed
```
- The release fence plus the acquire fence **synchronise through the atomic `ready`**. A fence with no atomic variable carrying the signal synchronises nothing in C++.
- Use it when **one fence covers many relaxed atomics** (e.g. poll several flags relaxed, then one acquire fence after any flag is seen). Otherwise prefer `store(release)` / `load(acquire)`.

#### 3.7.7 Embedded: devices and DMA [TI][QCOM]
```c
#define DMA_DOORBELL (*(volatile uint32_t *)0x40020010)  // MMIO: volatile ⇒ every access really happens
desc->addr = (uint32_t)buf;      // ordinary memory writes (descriptor the device will read)
desc->len  = n;
__DSB();                         // hardware: descriptor writes complete before the doorbell write
DMA_DOORBELL = 1;                // device starts reading desc
```
`volatile` handles the **compiler** layer; `__DMB()`/`__DSB()` handle the **hardware** layer; on cores with data caches the buffer may also need a cache clean/invalidate (note 15). Linux equivalents: `dma_wmb()`, `wmb()`.

#### 3.7.8 What to use when
| Situation | Use |
|---|---|
| Shared data structure with complex invariants | `std::mutex` |
| One flag / index / pointer handed between threads | `std::atomic` + release/acquire |
| Pure statistics counter | `std::atomic` + `relaxed` `fetch_add` |
| Store→load ordering (Dekker / Peterson) | `seq_cst` atomics or `atomic_thread_fence(seq_cst)` |
| Many relaxed atomics, one ordering point | `std::atomic_thread_fence` |
| ISR / signal handler on the same core | `volatile sig_atomic_t` / `atomic_signal_fence` |
| MMIO registers, DMA | `volatile` + `__DMB()`/`__DSB()` (or `wmb()`/`dma_wmb()`) |
| **Never** | `volatile` for inter-thread sync; hand-written asm fences in application code |

**Check yourself (ARM, multi-core): is the data → flag handoff correct?** (a) both `volatile` → ❌ (compiler stopped, hardware not). (b) `data = 42; asm volatile("" ::: "memory"); flag = 1;` → ❌ (same reason). (c) release fence + relaxed store / relaxed load + acquire fence (§3.7.6) → ✅. On x86, (a) and (b) *happen* to work (TSO), but they're still data races, which is the classic "works on x86, breaks on ARM" bug.

### 3.8 Lock-free SPSC ring buffer: walk through every line (HFT staple)
Single-Producer Single-Consumer: one thread only pushes, one thread only pops.
```cpp
template<typename T, size_t N>                 // N must be a power of two
class SpscQueue {
    alignas(64) std::atomic<size_t> head{0};   // next slot to READ  — written only by consumer
    alignas(64) std::atomic<size_t> tail{0};   // next slot to WRITE — written only by producer
    alignas(64) T buf[N];
public:
    bool push(const T& v) {                                      // producer thread
        size_t t = tail.load(std::memory_order_relaxed);         // (1)
        if (t - head.load(std::memory_order_acquire) == N)       // (2)
            return false;                                        //     full
        buf[t & (N - 1)] = v;                                    // (3)
        tail.store(t + 1, std::memory_order_release);            // (4) publish
        return true;
    }
    bool pop(T& v) {                                             // consumer thread
        size_t h = head.load(std::memory_order_relaxed);         // (5)
        if (h == tail.load(std::memory_order_acquire))           // (6)
            return false;                                        //     empty
        v = buf[h & (N - 1)];                                    // (7)
        head.store(h + 1, std::memory_order_release);            // (8) free the slot
        return true;
    }
};
```
| Line | Why this memory order |
|---|---|
| (1) `relaxed` load of own `tail` | only the producer writes `tail`, so it always sees its own latest value; no ordering needed |
| (4) `release` store of `tail` | the slot write (3) must become visible **before** the new tail. Otherwise the consumer could read a half-written slot |
| (6) `acquire` load of `tail` | pairs with (4): once the consumer sees the new tail, the slot data (3) is guaranteed visible, and (7) can't run early |
| (8) `release` store of `head` | the consumer's read of the slot (7) must finish **before** it hands the slot back |
| (2) `acquire` load of `head` | pairs with (8): the producer won't overwrite (3) a slot the consumer is still reading |
| `alignas(64)` on head, tail, buf | head is written by the consumer and tail by the producer; on one line they would **false-share** (§2.6) |
| `& (N-1)` | cheap modulo because N is a power of two; indices just keep increasing (unsigned wrap is fine) |

Why it's fast: **no locks, no RMW atomics** (only plain loads/stores plus ordering, which are free on x86), and no false sharing. A further optimisation: the producer keeps a **local cached copy of `head`** and re-reads the shared one only when the queue looks full, so the shared line moves far less often.

### 3.9 GPU coherence / consistency (brief) [NVIDIA]
- Each SM's L1 is generally **not hardware-coherent** with other SMs' L1s; **L2 is the point of coherence**.
- Weak memory model with **scopes**: `__threadfence_block()` (block), `__threadfence()` (device), `__threadfence_system()` (device + host); `cuda::atomic<T, cuda::thread_scope_device>`.
- `__syncthreads()` = execution barrier **plus** memory fence within a block.
- Interview line: "To pass data between blocks, write it, `__threadfence()`, then set a flag with an atomic. Readers must read the flag with an atomic or volatile access."

---

## 4. Worked Numericals / Traces

### N1. MESI trace (most-asked trace)
Three cores; line X starts **only in memory**. Fill the states after each operation.
| # | Operation | P1 | P2 | P3 | Interconnect | What happened |
|---|---|---|---|---|---|---|
| 0 | start | I | I | I | — | |
| 1 | P1 reads X | **E** | I | I | BusRd | miss; nobody else has X ⇒ memory supplies ⇒ Exclusive |
| 2 | P2 reads X | **S** | **S** | I | BusRd | P1 snoops "I have it in E" ⇒ E→S; P2 gets a copy in S |
| 3 | P1 writes X | **M** | I | I | BusUpgr | P1 is in S, so it can't write silently ⇒ invalidate P2 ⇒ M |
| 4 | P2 reads X | **S** | **S** | I | BusRd | P2 misses; P1 has it **dirty** (M) ⇒ P1 supplies data + **writes back to memory** ⇒ both S |
| 5 | P3 writes X | I | I | **M** | BusRdX | P3 misses on a write ⇒ RFO: get data + invalidate P1, P2 |
| 6 | P3 writes X | I | I | M | none | hit in M ⇒ silent |
**Totals:** 5 interconnect transactions (steps 1–5), **1** memory write-back (step 4).
*MOESI twist:* at step 4, P1 goes M→**O** (not S) and **no** write-back happens; P1 keeps supplying the dirty data.

**Try it:** same 3 cores, X only in memory. Sequence: P2 read, P2 write, P1 read, P3 read, P1 write. Give each state and count transactions and write-backs.
<details><summary>Answer</summary>

1. P2 read → P2 **E** (BusRd).
2. P2 write → P2 **M** (silent, no transaction).
3. P1 read → P2 supplies + writes back; P1 **S**, P2 **S** (BusRd, write-back #1).
4. P3 read → P1, P2, P3 all **S** (BusRd).
5. P1 write → BusUpgr; P1 **M**, P2 **I**, P3 **I**.

Total: **4** transactions, **1** write-back. Step 2 is free because of E: that's why E exists.
</details>

### N2. Lost-update count
Two threads each run `count++` (plain `int`, not atomic) **1000** times. What final values are possible?
- **Max = 2000** (no overlap). **Min = 2** (the famous worst case: an interleaving exists where almost every update is lost, with one load/store pair from each thread surviving at the very start/end). In practice you'll see values between 1000 and 2000, and different values on each run.
- With `std::atomic<int>` and `fetch_add` (or `++` on the atomic): always **2000**.
*Interview point: the answer "1000 is the minimum" is wrong. The minimum is 2. Be ready to explain the interleaving, or at least say "it can be far below 1000".*

### N3. False-sharing cost estimate
Two threads each do **10⁸** increments. Case A: counters on the same line. Case B: padded. Assume a line transfer ≈ **70 ns**, an L1-hit `lock add` ≈ **5 ns** (≈20 cycles at 4 GHz).
- **Case A (worst case):** every increment needs the line moved. Only one core can own the line at a time, so the transfers are **serialised**:
  2 × 10⁸ × 70 ns = **14 s**. (Real hardware usually manages a few increments per ownership, so you'd measure a few seconds, still terrible.)
- **Case B:** each core increments in its own L1, **in parallel**: 10⁸ × 5 ns = **0.5 s**.
- **Speedup:** 14 / 0.5 ≈ **28×** worst case. With plain non-atomic per-thread locals (~1 ns), it's ≈100×.

**Try it:** 4 threads, 10⁷ increments each, same line, 80 ns per transfer, vs padded at 4 ns per increment. Estimate both times.
<details><summary>Answer</summary>

Same line: 4 × 10⁷ × 80 ns = 3.2 s (serialised). Padded: 10⁷ × 4 ns = 0.04 s (the threads run in parallel). ≈ **80×**.
</details>

### N4. Litmus tests: x86 vs ARM
| Test | Code | Outcome asked | SC | x86 | ARM (no barriers) | Fix on ARM |
|---|---|---|---|---|---|---|
| Store buffer (Dekker) | P0: `x=1; r1=y` · P1: `y=1; r2=x` | r1=0 ∧ r2=0 | ✗ | **✓ allowed** | ✓ | full fence / seq_cst (also needed on x86!) |
| Message passing | P0: `data=1; flag=1` · P1: `r1=flag; r2=data` | r1=1 ∧ r2=0 | ✗ | ✗ | **✓ allowed** | release on flag store + acquire on flag load |
| Load buffering | P0: `r1=x; y=1` · P1: `r2=y; x=1` | r1=1 ∧ r2=1 | ✗ | ✗ | ✓ (allowed by the model) | acquire/release |
How to answer: find which reordering the outcome needs (Dekker needs Store→Load; MP needs Store→Store or Load→Load; LB needs Load→Store), then check the §3.4 table.

### N5. Directory storage overhead *(HW depth — skim)*
64 cores, full bit-vector directory (1 presence bit per core per line), 64 B lines.
Per line: 64 presence bits + ~2 state bits = 66 bits, against 64 × 8 = 512 data bits ⇒ 66/512 ≈ **12.9 %**.
At 1024 cores: 1026/512 ≈ **200 %** ⇒ big systems use sparse or limited-pointer directories.

---

## 5. Interview Questions

### Basic
- **Q: What is cache coherence and why is it needed?** Private write-back caches can hold stale copies. Coherence guarantees a single writer or many readers per line, and that reads see the latest write. Hardware does it automatically (§2.1–2.2).
- **Q: Coherence vs consistency?** Coherence: one address, everyone agrees on its history. Consistency: the order in which writes to **different** addresses become visible (§1.1, §3.1).
- **Q: Explain MESI. Why does E exist?** Table §2.4. E lets private data go read → write with no bus transaction (silent E→M).
- **Q: Snooping vs directory?** Broadcast vs a sharer list at a home node; directories scale (§2.3).

### Intermediate
- **Q [HFT][AMZN]: My multithreaded counter program gets slower with more threads. Why?** Either a contended atomic (the line migrates on every op) or false sharing between per-thread counters on one line. Fix: per-thread counters padded to 64/128 B, summed at the end.
- **Q [HFT]: What is false sharing? How do you detect and fix it?** §2.6. `perf c2c` HITM; `alignas(64/128)`; per-thread data.
- **Q: Does cache coherence make `count++` thread-safe?** No. Coherence orders single loads/stores, and `++` is load-add-store, so updates get lost (§2.7.1). Use `fetch_add`.
- **Q: Why is an atomic increment expensive under contention?** The line must be pulled in M state to each core in turn (~50–100 ns each), serialised. Plus a `lock` op is a full fence.
- **Q: Why is test-and-test-and-set better than test-and-set?** Waiters spin on a read (S copy, local L1, no traffic) instead of RFO-ing the line on every spin (§2.7.3).
- **Q: MESI vs MOESI?** O = dirty-but-shared; a remote read of a dirty line needs no memory write-back.

### Advanced / killer follow-ups
- **Q [HFT][DESHAW]: Dekker test. Can both loads read 0 on x86? Why? Fix?** Yes: each store waits in its store buffer while the later load reads the cache (Store→Load reordering, allowed by TSO). Fix: `mfence` / `xchg` / seq_cst atomics between the store and the load (§3.3).
- **Q: What is a memory fence? Is it hardware or compiler?** An ISA instruction (`mfence`, `dmb`, `fence`) that stops memory ops crossing it, by draining the store buffer and holding back later loads. A compiler barrier is a different thing: it stops only compiler reordering. Follow-up: *"When do you need `mfence` on x86?"* Only for store→load ordering (Dekker/Peterson, seq_cst stores). Message passing needs none on x86 (§3.5).
- **Q: Compiler barrier vs memory barrier?** A compiler barrier (`asm volatile("" ::: "memory")`, `atomic_signal_fence`) emits **no instruction** and stops only compiler reordering. A memory barrier is an ISA instruction (`mfence`, `dmb`) that stops CPU reordering, and with a `"memory"` clobber it is also a compiler barrier. `std::atomic` / `atomic_thread_fence` give both, emitting a hardware fence only if the ISA needs one (§3.7).
- **Q: On x86, what does `ready.store(true, memory_order_release)` compile to? Why write it?** A plain `mov`. It's needed to stop **compiler** reordering, and on ARM the same line becomes `stlr`.
- **Q: Why is an uncontended `lock add` ~20 cycles when the line is already in L1?** A locked op is a full fence on x86: the store buffer must drain first (§3.5.5).
- **Q: Is x86 sequentially consistent?** No, it's TSO: only Store→Load reordering, because of the store buffer.
- **Q: What does `memory_order_acquire` compile to on x86 vs ARM?** x86: a plain `mov` (TSO already orders it). ARM: `ldar`/`ldapr`. Follow-up: "So why use atomics on x86 at all?" To stop **compiler** reordering and register caching, and for atomic RMW.
- **Q: Why are seq_cst stores slow on x86?** The store must be visible before any later load ⇒ the store buffer must drain ⇒ `xchg`/`mfence`, tens of cycles or more.
- **Q: Is `volatile` enough for a flag between threads?** Not in C/C++: no atomicity, no ordering of other memory ops, still a data race (UB). Use `std::atomic`. `volatile` is for MMIO. (Java `volatile` *is* acquire/release.)
- **Q [HFT]: Design a lock-free SPSC queue. Justify each memory order.** §3.8 table.
- **Q: What is the ABA problem? How does LL/SC avoid it?** §2.7.4.
- **Q [APPLE]: How does Rosetta 2 run multithreaded x86 code correctly on ARM?** The M-series hardware TSO mode, enabled for translated processes.
- **Q [NVIDIA]: Are GPU L1 caches coherent?** Not with each other. L2 is the coherence point; use scoped fences/atomics.
- **Q: Your code works on x86 but breaks on ARM (or after porting to an Apple/Graviton server). Most likely cause?** It relied on TSO ordering: plain variables used as flags, or relaxed atomics where acquire/release was needed.

---

## 6. Common Traps
- Thinking coherence orders accesses to **different** variables. It doesn't; that's consistency.
- Thinking coherence makes `x++` atomic. It doesn't.
- "x86 is sequentially consistent." No: TSO (Store→Load can reorder). Dekker breaks on x86.
- Reasoning only about hardware and forgetting the **compiler** also reorders and caches in registers.
- Using `volatile` for thread synchronisation in C/C++.
- Padding to 64 B on Apple M (128 B lines), or ignoring Intel's adjacent-line prefetcher (pad to 128 B to be safe).
- Fixing only the producer side (release) on ARM and forgetting the consumer (acquire); both are needed.
- Saying the minimum of the 2-thread × 1000 `count++` race is 1000 (it is 2).
- Thinking the default `std::atomic` store is cheap on x86. It's seq_cst ⇒ `xchg`.

---

## 7. Quick Self-Check Quiz
1. Core 0 has line L in **E**, then writes it. How many interconnect messages? → **0** (silent E→M).
2. Two threads increment `a` and `b`, which are adjacent `long`s. Bug or slowdown? → **Slowdown** (false sharing), not a correctness bug if they're atomics.
3. On x86, can a later load be satisfied before an earlier store to a **different** address is visible? → **Yes** (store buffer, TSO).
4. On x86, can two stores become visible out of order? → **No** (FIFO store buffer). On ARM? → **Yes**.
5. What C++ orders make message passing correct, and where? → `release` on the flag store (producer), `acquire` on the flag load (consumer).

---

## 8. Revision Summary
| Topic | Remember |
|---|---|
| Coherence | ONE address: single writer or many readers + latest value; per **64 B line**; automatic in HW |
| Consistency | ORDER across DIFFERENT addresses; ISA contract (store buffer + OoO + compiler) |
| Snoop vs directory | broadcast vs sharer list at home node; directory scales |
| MESI | M only + dirty · E only + clean (silent → M) · S read-only shared · I invalid |
| Why E | private read-then-write needs no invalidate broadcast |
| MOESI / MESIF | O = dirty shared, no write-back (AMD) · F = one forwarder (Intel) |
| Line transfer cost | ~1 ns L1 hit vs **~50–100 ns** from another core |
| False sharing | different vars, same line → ping-pong → `alignas(64/128)`, `perf c2c` |
| `count++` | coherence ≠ atomicity → lost updates; use `fetch_add` (`lock xadd`) |
| Atomic RMW | own line in M + lock it; ~20 cycles uncontended, 100s contended; full fence on x86 |
| Spinlock | TTAS: spin on a read (S, local), then exchange; `pause` |
| SC | one interleaving, program order; Dekker (0,0) impossible |
| Store buffer | makes Store→Load reordering visible → Dekker (0,0) possible on x86 and ARM |
| x86 TSO | only Store→Load; acquire/release free; seq_cst store = `xchg` |
| ARM / RISC-V | weak: all 4 reorderings; `ldar`/`stlr`/`dmb` |
| Fence | ISA instruction: earlier mem ops visible before later ones run (drains the store buffer); x86 `mfence`/`lock`, ARM `dmb`, RISC-V `fence`; not needed for single-thread code |
| Acquire / release | one-way fences; release publishes, acquire subscribes; must pair |
| Two layers | compiler (as-if rule; stopped by `asm("":::"memory")`, `atomic_signal_fence`; no instruction) + hardware (stopped by ISA fences); `std::atomic`/`atomic_thread_fence`/mutex stop both |
| C/C++ fence tools | mutex > `std::atomic` + order > `atomic_thread_fence` > builtins `__atomic_*` > intrinsics `_mm_mfence`/`__DMB` > asm with `"memory"` |
| `volatile` | MMIO yes, threads no (C/C++); Java volatile = acq/rel |
| SPSC queue | relaxed own index, acquire other's index, release after slot access, pad head/tail |
| GPU | L1s not coherent; L2 is the coherence point; scoped fences |
