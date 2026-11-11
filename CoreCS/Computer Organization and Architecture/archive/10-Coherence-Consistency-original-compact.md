# 10. Cache Coherence & Memory Consistency ⭐ (HFT favourite)

> **Asked at:** [HFT] [DESHAW] false sharing, atomics cost, lock-free queues, memory ordering, `std::atomic` memory orders · [APPLE] [QCOM] [NVIDIA] MESI/MOESI, snooping vs directory, ARM weak ordering, GPU memory model · [AMZN] concept checks.
> **Prerequisites:** [06 Cache](06-Cache-Memory.md), [09 OoO](09-ILP-Superscalar-Out-of-Order.md).

---

## 1. Intuition
Each core has a private cache. If core A caches `x = 0` and core B writes `x = 1`, A must not keep reading 0 forever. **Coherence** = rules for *one address*: all cores agree on the order of writes to it. **Consistency** = rules for *different addresses*: in what order can other cores observe my writes to `x` and `y`? Coherence is a property of caches; consistency is a property of the whole memory system (store buffers, OoO, compiler) and is part of the ISA contract.

---

## 2. Deep Dive — Coherence

### 2.1 Coherence invariants
1. **SWMR (single writer, multiple readers):** at any time, a line is either writable by exactly one core or read-only by any number.
2. **Data-value invariant:** a read returns the value of the last write in the line's serialized write order.
Implemented with per-line states and a protocol over an interconnect.

### 2.2 Snooping vs directory
| | Snooping | Directory |
|---|---|---|
| How | every miss is **broadcast** on a shared bus/ring; all caches snoop | a **home node** tracks sharers per line; point-to-point messages |
| Latency | low (one broadcast) | extra indirection (requester → home → owner) |
| Scalability | poor (broadcast BW) | good (many cores, multi-socket) |
| Used in | small multicores, historically | many-core servers, multi-socket (ccNUMA), with **snoop filters** |
Modern chips are hybrids: an inclusive LLC or snoop filter acts like a directory to avoid broadcasting.

### 2.3 MESI protocol
| State | Meaning | Clean/dirty | Other copies? |
|---|---|---|---|
| **M**odified | only valid copy, changed | dirty | none |
| **E**xclusive | only copy, unchanged | clean | none |
| **S**hared | read-only copy | clean | maybe |
| **I**nvalid | not present/usable | — | — |

Transitions (local core = P, others = remote):
| Current | Event | Action | Next |
|---|---|---|---|
| I | P read, no other sharer | BusRd | **E** |
| I | P read, others have it | BusRd (data from cache/memory) | **S** |
| I | P write | **BusRdX** (read-for-ownership, invalidate others) | **M** |
| S | P write | **BusUpgr / invalidate** others | **M** |
| E | P write | none — **silent upgrade** (why E exists) | **M** |
| M | remote read | supply data, write back | **S** |
| M | remote write (RdX) | supply data | **I** |
| E | remote read | | **S** |
| S/E | remote write | | **I** |
- **Why E?** Private data (most data!) can be read then written without any bus transaction.
- **MOESI (AMD):** adds **O**wned — dirty but shared; the owner supplies data, no write-back to memory on remote read.
- **MESIF (Intel):** adds **F**orward — one designated sharer answers requests (avoids multiple responders).
- **Write-invalidate** (all modern protocols) vs **write-update** (broadcast new value — more traffic, rare).

### 2.4 False sharing [HFT][DESHAW][NVIDIA]
Two threads write **different variables on the same 64 B line** ⇒ the line ping-pongs in M state between cores (each write invalidates the other's copy) — no logical sharing, yet ~50–100+ ns per transfer.
```cpp
struct Counters { std::atomic<long> a; std::atomic<long> b; };          // same line → false sharing
struct alignas(64) Padded { std::atomic<long> v; };                       // one per line
Padded counters[NTHREADS];
// C++17: std::hardware_destructive_interference_size (usually 64; 128 on Apple M / some ARM)
```
Can cause 10×+ slowdowns. Note: Intel's adjacent-line (spatial) prefetcher pairs 128 B; many HFT codebases pad to **128 B**.

### 2.5 Atomics and locks — what they cost
- Atomic RMW (`lock xadd`, `lock cmpxchg`, ARM `LDADD`/`CAS` or LL/SC `LDXR/STXR`): the core obtains the line in **M** and holds it during the operation. Uncontended: ~10–25 cycles on x86 (also acts as a full fence). Contended across cores: hundreds of cycles per op (line migration).
- **Spinlock pitfalls:** `while (test_and_set(lock))` hammers the line with RFOs. **Test-and-test-and-set**: spin on a *read* (stays in S locally), only try the atomic when it looks free. Add `pause`/`yield`. Scalable: ticket locks (fairness), **MCS/CLH** queue locks (each waiter spins on its own line).
- **LL/SC (load-linked / store-conditional):** SC fails if the line was touched since LL — avoids ABA by construction (spurious failures possible).
- **ABA problem** with CAS: value changes A→B→A; fix with tagged pointers / version counters / hazard pointers.

---

## 3. Deep Dive — Memory Consistency

### 3.1 Sequential Consistency (SC) — Lamport
"The result is as if all operations of all cores executed in **some single interleaving**, each core's operations appearing **in program order**." Intuitive, but forbids common hardware optimisations (store buffers, OoO loads).

### 3.2 Why real hardware breaks SC: the store buffer
A store sits in the core's **store buffer** until it commits to L1 (needs the line in M). Loads proceed meanwhile and can bypass older stores (to different addresses).
**Store-buffer litmus test (Dekker):**
```
 Initially x = y = 0
 Core 0:            Core 1:
   x = 1;             y = 1;
   r1 = y;            r2 = x;
 Can r1 == 0 && r2 == 0 ?   SC: NO.   x86 (TSO): YES!   ARM: YES.
```
Both stores wait in store buffers while both loads read 0 from caches. Fix: a **full fence** (`mfence`, or a `lock`ed op / `xchg`; ARM `dmb ish`) between the store and the load — or `std::atomic` with `memory_order_seq_cst`.

### 3.3 Models of real ISAs
| Reordering allowed? | SC | **x86 (TSO)** | **ARMv8 / RISC-V (RVWMO)** / POWER |
|---|---|---|---|
| Load → Load | no | no | **yes** |
| Load → Store | no | no | **yes** |
| Store → Store | no | no | **yes** |
| **Store → Load** | no | **yes** (store buffer) | **yes** |
| Multi-copy atomic | yes | yes | ARMv8: yes ("other-multi-copy atomic"); POWER: no |
x86 is "strong": only store→load reordering. ARM is weak: needs explicit barriers/acquire-release for publication patterns. Apple M-series silicon has a hardware TSO mode used by Rosetta 2 to run x86 code correctly — great trivia for [APPLE].

### 3.4 Fences, acquire/release, and C++ atomics
| C++ `memory_order` | Guarantee | x86 cost | ARMv8 |
|---|---|---|---|
| `relaxed` | atomicity only, no ordering | plain mov | plain ldr/str |
| `acquire` (loads) | later ops can't move **before** it | plain mov (TSO gives it free) | `LDAR` / `LDAPR` |
| `release` (stores) | earlier ops can't move **after** it | plain mov | `STLR` |
| `acq_rel` (RMW) | both | `lock` prefix | `LDADDAL` etc. |
| `seq_cst` | single total order of all seq_cst ops | **store = `xchg`/`mov+mfence`** (expensive) | `STLR`/`LDAR` (ARMv8 designed for this) |

**Message-passing pattern (publish data):**
```cpp
// Producer                                  // Consumer
data = 42;                                   while (!ready.load(std::memory_order_acquire)) ;
ready.store(true, std::memory_order_release);   assert(data == 42);   // guaranteed
```
Release "publishes" everything written before it; acquire "subscribes".

**Compiler reordering** is a separate hazard: without atomics/fences the compiler may hoist, sink, or cache values in registers. `volatile` prevents compiler caching (needed for **MMIO** in embedded [TI]) but is **not** a synchronisation primitive in C/C++ (no atomicity, no hardware ordering).

### 3.5 Lock-free SPSC ring buffer (HFT staple)
```cpp
template<typename T, size_t N>                 // N power of two
class SpscQueue {
    alignas(64) std::atomic<size_t> head{0};   // written by consumer
    alignas(64) std::atomic<size_t> tail{0};   // written by producer
    alignas(64) T buf[N];
public:
    bool push(const T& v) {
        size_t t = tail.load(std::memory_order_relaxed);
        if (t - head.load(std::memory_order_acquire) == N) return false;   // full
        buf[t & (N - 1)] = v;
        tail.store(t + 1, std::memory_order_release);                       // publish
        return true;
    }
    bool pop(T& v) {
        size_t h = head.load(std::memory_order_relaxed);
        if (h == tail.load(std::memory_order_acquire)) return false;        // empty
        v = buf[h & (N - 1)];
        head.store(h + 1, std::memory_order_release);
        return true;
    }
};
```
Why fast: no locks, no RMW atomics (only loads/stores), head/tail on separate lines (no false sharing). Further: cache the other side's index locally to avoid reading the shared line every call.

### 3.6 GPU coherence/consistency (brief) [NVIDIA]
- Per-SM L1 caches are generally **not hardware-coherent** with each other; L2 is the point of coherence.
- Weak memory model; scopes: `__threadfence_block()`, `__threadfence()` (device), `__threadfence_system()`; `cuda::atomic` with `cuda::thread_scope_*`.
- `__syncthreads()` = barrier + memory fence within a block.

---

## 4. Worked Numericals / Traces

**N1. MESI trace.** Three cores, line X initially only in memory.
| Op | P1 | P2 | P3 | Bus transaction |
|---|---|---|---|---|
| P1 read X | **E** | I | I | BusRd (memory supplies) |
| P2 read X | S | **S** | I | BusRd (P1 may supply) |
| P1 write X | **M** | I | I | BusUpgr (invalidate) |
| P2 read X | S | **S** | I | BusRd; P1 supplies data + writes back (MOESI: P1 → **O**, no write-back) |
| P3 write X | I | I | **M** | BusRdX |
| P3 write X | I | I | M | none (hit in M) |
Total: 4 bus transactions; 1 memory write-back under MESI.

**N2. Directory overhead.** 64 cores, full bit-vector directory, 64 B lines: 64 presence bits + ~2 state bits per line = 66 bits per 512 data bits ≈ **12.9%**. At 1024 cores: 200% ⇒ need limited-pointer or coarse vectors / sparse directories.

**N3. False sharing cost estimate.** Two threads each do 10⁸ increments on counters in the same line. If each increment needs a line transfer of ~70 ns: ~7 s. Padded: each core increments in its own L1 at ~1 ns (with atomics ~5 ns) ⇒ ~0.1–0.5 s ⇒ **~15–70× faster**.

**N4. Litmus test outcomes.** Message passing on ARM *without* barriers:
```
 P0: data = 1; flag = 1;      P1: r1 = flag; r2 = data;
```
Can r1 = 1, r2 = 0? **x86: no** (store→store and load→load preserved). **ARM: yes** (both reorderings allowed) — needs `STLR` on flag and `LDAR` on flag (release/acquire).

---

## 5. Interview Questions

### Basic
- **Q: What is cache coherence and why is it needed?** (1, 2.1.)
- **Q: Explain MESI states. Why does E exist?** (2.3.)
- **Q: Snooping vs directory?** (2.2.)

### Intermediate
- **Q [HFT]: What is false sharing, how do you detect and fix it?** Same-line writes by different cores; detect with `perf c2c` (HITM events) / VTune; fix with padding/alignment, per-thread data, batching.
- **Q: Coherence vs consistency?** Single address vs ordering across addresses.
- **Q: Why is an atomic increment expensive under contention?** Line must migrate in M state to each incrementing core; serialised ownership transfers.
- **Q: What's the difference between MOESI and MESI?** O state lets dirty data be shared without writing back.

### Advanced / killer follow-ups
- **Q [HFT][DESHAW]: In the Dekker example, can both read 0 on x86? Why? Fix?** Yes, store buffer (TSO); `mfence` / seq_cst.
- **Q: What does `memory_order_acquire` compile to on x86 vs ARM?** x86: plain load (TSO already orders it); ARM: `LDAR`/`LDAPR`.
- **Q: Is `volatile` enough for a flag between threads?** No (C/C++): no atomicity guarantees, no ordering w.r.t. other memory, data race = UB. Use `std::atomic`. (Java `volatile` is different — it has acquire/release semantics.)
- **Q: Design a lock-free SPSC queue.** (3.5) — explain each memory order and the padding.
- **Q: Why might `seq_cst` stores be slow on x86?** Need a full fence (`xchg`/`mfence`) to drain the store buffer before later loads.
- **Q [APPLE]: How does Rosetta 2 run x86 multithreaded code correctly on ARM?** Apple cores implement an optional TSO mode for translated processes.
- **Q [NVIDIA]: Are GPU L1 caches coherent?** Not with each other in hardware; use fences/atomics at the right scope; L2 is the coherence point.

---

## 6. Common Traps
- Thinking coherence guarantees ordering across different variables.
- "x86 is sequentially consistent" — no, TSO (store→load reordering).
- Forgetting compiler reordering when reasoning only about hardware.
- Using `volatile` for thread synchronisation in C++.
- Padding to 64 B on Apple (128 B lines) or ignoring adjacent-line prefetch.

---

## 7. Revision Summary
| | |
|---|---|
| Coherence | SWMR + data-value invariant, per address |
| MESI | E = silent upgrade; M→S on remote read (write back) |
| MOESI / MESIF | O = dirty shared (AMD); F = forwarder (Intel) |
| Snoop vs directory | broadcast vs home node; directory scales |
| False sharing | same line, different vars → pad to 64/128 B |
| Atomics | own line in M; ~20 cycles uncontended, 100s contended |
| SC | one interleaving, program order |
| x86 TSO | only store→load reordering (store buffer) |
| ARM/RISC-V | weak; use acquire/release/dmb |
| C++ | relaxed / acquire / release / acq_rel / seq_cst |
| volatile | MMIO yes, threads no |
