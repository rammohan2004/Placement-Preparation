# 14. Low-Latency Hardware Awareness ("Mechanical Sympathy") for HFT

> **Asked at:** **[HFT]** IMC, Optiver, Tower, Graviton, Quadeye, Jane Street (systems roles) · **[DESHAW]** systems/C++ rounds · also [NVIDIA] perf engineering and [AMZN] high-performance services.
> This note **connects** all earlier notes to the question HFT interviewers really ask: *"Here's some code / a system — why is it slow, and how would you make it faster and more predictable?"*
> **Prerequisites:** [05](05-Branch-Prediction.md), [06](06-Cache-Memory.md), [07](07-Virtual-Memory-and-TLB.md), [09](09-ILP-Superscalar-Out-of-Order.md), [10](10-Cache-Coherence-and-Memory-Consistency.md), [11](11-SIMD-Multithreading-Multicore.md), [13](13-IO-Interrupts-DMA.md).

---

## 1. Intuition
In trading, the fastest correct reaction to a market event wins. **Tick-to-trade** for software systems is on the order of single-digit microseconds; FPGA systems go to tens–hundreds of nanoseconds. At that scale, **one DRAM miss (~100 ns), one page fault (~1 µs), or one context switch (~several µs) is a disaster**. And not just average latency — **jitter (p99, p99.9)** loses money. Mechanical sympathy = writing software that works *with* caches, predictors, TLBs and cores instead of against them.

---

## 2. Deep Dive

### 2.1 Latency numbers to memorise (order of magnitude, modern x86 server)
| Event | Approx. |
|---|---|
| L1 hit | ~1 ns (4–5 cycles) |
| Branch mispredict | ~5 ns (15–20 cycles) |
| L2 hit | ~3–5 ns |
| L3 hit | ~10–20 ns |
| Uncontended atomic / lock | ~5–25 ns |
| Cross-core cache-line transfer | ~40–100 ns |
| DRAM (local NUMA) | ~80–100 ns |
| DRAM (remote NUMA) | ~130–200 ns |
| System call (simple) | ~0.1–1 µs |
| Minor page fault | ~0.5–2 µs |
| PCIe round trip | ~0.5–1 µs |
| Context switch | ~1–5 µs (plus cache pollution) |
| Kernel UDP receive path | ~5–10 µs |
| Kernel-bypass NIC wire-to-app | ~1–2 µs (FPGA: < 0.1 µs) |
| Switch hop (cut-through) | ~0.1–0.5 µs |
| Light in fibre | ~5 µs/km (microwave/air ~3.3 µs/km) |

### 2.2 Where the time goes in a software trading path
```
 NIC (wire) → DMA into ring → [kernel bypass] user-space poll → decode feed (parse, endianness)
   → update order book → strategy/pricing → risk checks → encode order → NIC send (DMA / PIO)
```
Every stage must be: **cache-resident, branch-predictable, allocation-free, syscall-free, lock-free, and on an isolated core.**

### 2.3 The principles (each maps to a COA concept)
| # | Principle | COA reason | Techniques |
|---|---|---|---|
| 1 | **Stay in cache** | DRAM is 100× L1 ([06](06-Cache-Memory.md)) | compact structs, SoA, hot/cold split, flat arrays, small integer types, avoid pointer chasing |
| 2 | **Keep caches warm** | idle periods → hot path evicted by other work | "cache warming": run the full path on dummy data continuously (don't send), pin, avoid noisy neighbours (Intel CAT) |
| 3 | **Predictable branches** | mispredict = 15–20 cycles ([05](05-Branch-Prediction.md)) | branchless code, `[[likely]]`, move rare checks out of the loop, templates/CRTP instead of virtual dispatch |
| 4 | **No allocation in hot path** | `malloc` = locks/syscalls/page faults ([07](07-Virtual-Memory-and-TLB.md)) | preallocate, object pools, arenas, fixed-capacity containers |
| 5 | **No syscalls / page faults** | traps, TLB/cache pollution | pre-fault (`MAP_POPULATE`, touch memory), `mlockall`, huge pages, kernel-bypass I/O, vDSO time |
| 6 | **No locks; careful sharing** | coherence traffic ([10](10-Cache-Coherence-and-Memory-Consistency.md)) | SPSC ring buffers, single-writer principle, padding to 64/128 B, acquire/release only |
| 7 | **Own the core** | scheduler, interrupts, SMT siblings add jitter ([11](11-SIMD-Multithreading-Multicore.md)) | `isolcpus`, `nohz_full`, pinning, IRQ affinity away, SMT off, busy-poll |
| 8 | **Stable frequency** | P-/C-state transitions take µs | disable deep C-states, fixed frequency, BIOS performance profile |
| 9 | **NUMA locality** | remote memory/PCIe 1.5–2× slower | NIC, thread, and memory on the same socket |
| 10 | **Parallelise across independent work, not on the critical path** | Amdahl; handoffs cost ~100 ns | single-threaded hot path, offload logging/persistence to other cores via queues |
| 11 | **Exploit ILP/SIMD** | OoO and vector units ([09](09-ILP-Superscalar-Out-of-Order.md)) | break dependency chains, SIMD parsing (e.g., find delimiters with AVX2) |
| 12 | **Measure, don't guess** | | `rdtsc`, `perf stat`, `perf c2c`, VTune, latency histograms |

### 2.4 Code patterns

**Branchless & predictable**
```cpp
// Bad: unpredictable branch in hot loop
if (side == Side::Buy) best = std::max(best, px); else best = std::min(best, px);
// Better: separate books per side (no branch), or branchless select compiled to cmov
best = (px > best) ? px : best;
```

**Avoid virtual dispatch in the hot path (static polymorphism)**
```cpp
template <typename Derived>
struct Strategy {
    void onTick(const Tick& t) { static_cast<Derived*>(this)->onTickImpl(t); }  // inlinable
};
struct MarketMaker : Strategy<MarketMaker> { void onTickImpl(const Tick& t); };
```
A virtual call itself is cheap (~1–3 ns) when predicted; the real costs are **no inlining** (lost optimisation) and **mispredicts** when the call site is polymorphic.

**Order book layout (array-indexed price levels)**
```cpp
// Prices are integer ticks (fixed-point, see note 01) within a known band.
struct Level { int64_t qty; uint32_t count; uint32_t pad; };   // 16 B → 4 levels per cache line
struct Book {
    alignas(64) Level bids[MAX_TICKS];       // index = price_tick - base_tick  → O(1), contiguous
    alignas(64) Level asks[MAX_TICKS];
    int32_t bestBid, bestAsk;                // maintained incrementally
};
// vs std::map<Price, Level>: red-black tree → each lookup = several dependent cache misses.
```

**Timing with the TSC**
```cpp
#include <x86intrin.h>
static inline uint64_t tsc_start() { _mm_lfence(); return __rdtsc(); }     // fence: don't start early
static inline uint64_t tsc_stop()  { unsigned aux; uint64_t t = __rdtscp(&aux); _mm_lfence(); return t; }
// Requires invariant TSC; convert cycles → ns with calibrated frequency; record into a histogram, report p50/p99/p99.9/max.
```

**Software prefetch (when the next address is known early)**
```cpp
for (size_t i = 0; i < n; ++i) {
    __builtin_prefetch(&table[keys[i + 8]]);   // prefetch ~8 iterations ahead
    process(table[keys[i]]);
}
```

### 2.5 Measuring performance (what tools tell you)
| Tool / counter | Tells you |
|---|---|
| `perf stat -e cycles,instructions,branch-misses,cache-misses,LLC-load-misses,dTLB-load-misses` | IPC, mispredict rate, miss rates |
| `perf record` / flame graphs | where the time goes |
| `perf c2c` | **false sharing / HITM** contention |
| Intel VTune "Top-down" (front-end bound, bad speculation, back-end memory/core bound, retiring) | which pipeline bottleneck dominates |
| Latency histograms (HdrHistogram) | tails, not averages |
**Micro-benchmark pitfalls:** dead-code elimination (use `benchmark::DoNotOptimize`), cold vs warm caches, CPU frequency ramp, measuring overhead of the timer itself, too few samples for tails, unrepresentative data (branch predictor learns your synthetic pattern!).

### 2.6 FPGA & hardware acceleration (awareness)
FPGAs parse market data and fire orders in hardware (deterministic, ~10s–100s of ns), with software setting parameters. Interviews for HFT hardware roles ask Verilog, pipelining in RTL, clock-domain crossing, and timing closure — see Digital Logic notes (coming later).

---

## 3. Worked Numericals

**N1. Message budget.** A feed peaks at 10 million messages/s on one core ⇒ budget = **100 ns/message** ≈ 300 cycles at 3 GHz ⇒ **at most one DRAM miss per message**, and zero syscalls/allocations.

**N2. `std::map` vs array book.** Book of 1000 levels in a red-black tree: depth ≈ log₂1000 ≈ 10 nodes; if half miss to L3 (~15 ns) ⇒ ~75 ns per lookup. Array-indexed level: 1 access, likely L1/L2 ⇒ ~2–5 ns ⇒ **~15–30× faster**.

**N3. Mispredict cost.** A hot loop processing 10⁷ msgs/s has a 50/50 unpredictable branch: 0.5 × 17 cycles ≈ 8.5 cycles/msg ≈ 2.8 ns at 3 GHz ⇒ **~3% of the 100 ns budget** per such branch; five of them ≈ 14%.

**N4. Page fault on first touch.** Lazy allocation of a 64 MB buffer with 4 KB pages: 16,384 first-touch minor faults × ~1 µs ≈ **16 ms** of stalls scattered through trading — pre-fault at startup. With 2 MB huge pages: 32 faults.

**N5. Cross-core handoff.** Passing each message from the feed thread to a strategy thread via a queue costs ~1 cache-line transfer (~70 ns) + possibly another for the index ⇒ ~100+ ns, i.e., the **entire** budget from N1 ⇒ keep the critical path on one core, or batch.

---

## 4. Interview Questions

### Intermediate
- **Q: What are the most important latency numbers?** (2.1.)
- **Q: Why is `std::list` / `std::map` bad for low latency? What would you use?** Pointer chasing → cache misses; use `std::vector`, flat/open-addressing hash maps, array-indexed structures, B-tree-like wide nodes.
- **Q: What's the cost of a virtual function call?** Indirect branch (cheap if predicted) + no inlining; costly when polymorphic at the call site.
- **Q: Why avoid `new`/`malloc` in the hot path?** Unbounded latency: locks, fragmentation search, `brk`/`mmap` syscalls, page faults on first touch.
- **Q: How do you measure the latency of a function precisely?** TSC with fences, many iterations, warm caches (and also cold, if relevant), histogram, subtract timer overhead, pin core, fixed frequency.

### Advanced / killer follow-ups
- **Q: Design a low-latency order book.** Integer ticks; array-indexed levels around the current price (re-centre when needed); per-level FIFO of orders with an intrusive list in a preallocated pool; hash map order-id → order (open addressing); maintain best bid/ask incrementally; separate bid/ask arrays (no side branches); cache-line-aligned.
- **Q: Your strategy has p50 = 2 µs, p99.9 = 80 µs. How do you find the cause?** Correlate spikes with events: interrupts (`/proc/interrupts`), context switches (`perf sched`), page faults (`perf stat -e page-faults`), TLB shootdowns, SMIs (MSR counters), frequency changes, GC/logging, cache misses after idle. Fix with isolation/pinning, pre-faulting, huge pages, warming, removing syscalls.
- **Q: Two threads each increment their own counter and it's slower than one thread. Why?** False sharing ([10](10-Cache-Coherence-and-Memory-Consistency.md)).
- **Q: Why turn off hyper-threading / deep C-states / turbo variability?** Shared resources & wake-up latency & frequency transitions add jitter.
- **Q: How would you pass messages between two threads with minimal latency?** SPSC ring buffer, padded indices, acquire/release, busy-polling consumer, cached copies of the remote index, batching where allowed.
- **Q: Why might adding a `printf`/log line in the hot path cost microseconds?** Formatting, locks, `write` syscall, I/O — use an asynchronous logger: write binary args into an SPSC queue, format on another core.
- **Q [DESHAW]: Explain why the sorted-array loop runs faster and two ways to fix the unsorted case.** Branch prediction; branchless arithmetic / cmov / SIMD.

---

## 5. Common Traps
- Optimising the average and ignoring the tail.
- Benchmarks that let the compiler delete the work or that run on warmed predictors with unrealistic data.
- Adding threads to a latency-critical path (handoff cost > work).
- Using floating point for prices; using `volatile` instead of atomics.

---

## 6. Revision Summary
- Budget thinking: 100 ns ≈ 300 cycles ≈ one DRAM miss.
- Stay in cache, warm it, avoid pointer chasing, compact SoA layouts.
- Predictable branches, no virtual dispatch on polymorphic hot sites.
- No malloc, no syscalls, no page faults (pre-fault, mlock, huge pages).
- Lock-free SPSC + padding; single-writer principle.
- Isolated, pinned cores; SMT off; fixed frequency; NUMA-local NIC/memory.
- Kernel bypass + busy polling.
- Measure with TSC + histograms; perf / VTune top-down / perf c2c.
