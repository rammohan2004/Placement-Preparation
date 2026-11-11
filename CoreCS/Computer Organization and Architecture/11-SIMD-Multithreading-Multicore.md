# 11. SIMD, Multithreading & Multicore

> **Asked at:** [NVIDIA] Flynn's taxonomy, SIMD vs SIMT, peak-FLOPs math · [HFT] [DESHAW] SMT on/off, NUMA, core pinning, auto-vectorisation · [QCOM] [APPLE] big.LITTLE / heterogeneous cores, NEON/SVE · [AMZN] multicore basics.
> **Prerequisites:** [09 OoO](09-ILP-Superscalar-Out-of-Order.md), [10 Coherence](10-Cache-Coherence-and-Memory-Consistency.md).

---

## 1. Intuition
After ILP ran out of steam, hardware found parallelism at two other levels:
- **Data-level parallelism (DLP):** do the *same* operation on many data elements at once → **SIMD** vectors, GPUs.
- **Thread-level parallelism (TLP):** run *independent* instruction streams → **multithreading** (share one core) and **multicore** (many cores).

---

## 2. Deep Dive

### 2.1 Flynn's taxonomy
| | Single data | Multiple data |
|---|---|---|
| **Single instruction** | SISD — classic uniprocessor | **SIMD** — vector units (AVX, NEON), GPUs (as SIMT) |
| **Multiple instruction** | MISD — rare (fault-tolerant pipelines, systolic arrays debated) | **MIMD** — multicore, clusters |
GPU = **SIMT** (single instruction, multiple threads): SIMD hardware with a per-thread programming model ([12](12-GPU-Architecture.md)).

### 2.2 SIMD on CPUs
| ISA | Width | Notes |
|---|---|---|
| SSE (x86) | 128-bit | 4 floats |
| AVX / AVX2 | 256-bit | 8 floats, FMA, gather |
| AVX-512 | 512-bit | 16 floats, **mask registers**, scatter; may lower clock on older Intel |
| ARM NEON | 128-bit | all Cortex-A / Apple |
| ARM **SVE/SVE2** | 128–2048-bit, **vector-length agnostic** | same binary runs on any width (Fujitsu A64FX, Neoverse V-series) |
| RISC-V V | VLA like SVE | |

```cpp
#include <immintrin.h>
void add(float* __restrict a, const float* __restrict b, const float* __restrict c, int n) {
    int i = 0;
    for (; i + 8 <= n; i += 8) {
        __m256 vb = _mm256_loadu_ps(b + i);
        __m256 vc = _mm256_loadu_ps(c + i);
        _mm256_storeu_ps(a + i, _mm256_add_ps(vb, vc));
    }
    for (; i < n; ++i) a[i] = b[i] + c[i];   // scalar tail
}
```
**What blocks auto-vectorisation:**
- Possible **pointer aliasing** (use `__restrict`, or the compiler adds runtime checks).
- **Loop-carried dependences** (`a[i] = a[i-1] + x`).
- Function calls, complex control flow (branches → need masking/blend).
- Non-unit strides / AoS layouts (need gathers) → prefer **SoA**.
- FP reductions without `-ffast-math` / `-fassociative-math` (reassociation changes results).
- Check with `-fopt-info-vec` (GCC) / `-Rpass=loop-vectorize` (Clang).

### 2.3 Hardware multithreading (one core, many threads)
| Type | Switch | Hides | Example |
|---|---|---|---|
| Coarse-grained | on long stalls (L3 miss) | long latencies | old Itanium Montecito |
| Fine-grained | every cycle, round-robin | short + long stalls; single-thread perf suffers | Sun Niagara, **GPU warp schedulers** |
| **SMT** (simultaneous) | instructions from multiple threads issue **in the same cycle** on an OoO core | fills empty issue slots | Intel Hyper-Threading, AMD SMT (2-way), IBM POWER (8-way) |
SMT duplicates architectural state (PC, registers, RAT) but **shares** caches, TLBs, execution units, branch predictors, ROB/queues (partitioned or shared).
- Throughput gain typically ~0–30%.
- **[HFT]: SMT is often disabled** (or the sibling hyperthread left idle): the sibling competes for L1/L2, ports and TLB ⇒ jitter and higher tail latency for the critical thread.
- Security: shared structures ⇒ side channels (e.g., PortSmash, L1TF/MDS mitigations disable SMT for untrusted workloads).

### 2.4 Multicore organisation
- Private L1/L2 per core, shared **sliced LLC**, connected by a **ring** (client CPUs) or **mesh** (servers) or chiplets via **Infinity Fabric** (AMD CCDs with per-CCD L3).
- Coherence across cores ([10](10-Cache-Coherence-and-Memory-Consistency.md)).
- **Heterogeneous (big.LITTLE / DynamIQ / Apple P- and E-cores / Intel P/E):** big OoO cores for latency, small efficient cores for background tasks ⇒ performance per watt. Scheduler must place threads well (Intel Thread Director, Linux EAS).

### 2.5 NUMA [HFT][DESHAW]
Multi-socket (and large chiplet) systems: each socket has its own memory controllers.
| Access | Typical latency |
|---|---|
| Local DRAM | ~80–100 ns |
| Remote DRAM (other socket) | ~130–200 ns |
- Linux **first-touch** policy: a page is allocated on the node of the thread that first writes it ⇒ initialise data on the thread that will use it.
- Tools: `numactl --cpunodebind --membind`, `libnuma`, `lscpu`, `lstopo`.
- [HFT] Pin the trading thread and its NIC's interrupts/queues to the **same NUMA node as the NIC's PCIe slot**.

### 2.6 Core isolation & affinity (systems knowledge HFT loves)
`isolcpus`/`nohz_full`/`rcu_nocbs` kernel params, `taskset`/`pthread_setaffinity_np`, IRQ affinity, busy-polling instead of sleeping, disabling C-states and frequency scaling (fixed max frequency) ⇒ predictable latency.

### 2.7 Synchronisation cost scale (rough orders of magnitude)
| Operation | Cost |
|---|---|
| L1 hit | ~1 ns |
| Uncontended atomic RMW | ~5–10 ns |
| Cross-core cache-line transfer | ~40–100 ns |
| Uncontended mutex lock/unlock | ~20–50 ns |
| Contended mutex (futex syscall, context switch) | ~1–10 µs |
| Thread creation | ~10–50 µs |

---

## 3. Worked Numericals

**N1. Peak FLOPs of one core.** 3.0 GHz, 2 FMA units, AVX-512 (16 floats), FMA = 2 FLOPs:
3×10⁹ × 2 × 16 × 2 = **192 GFLOP/s (FP32)**. 32 cores ⇒ 6.1 TFLOP/s.

**N2. Vectorisation speedup bound.** A loop is 70% of runtime and is vectorised 8-wide (perfect). Amdahl: 1 / (0.3 + 0.7/8) = **2.58×**. If the loop is memory-bound, maybe only ~1.2× — SIMD doesn't add bandwidth.

**N3. NUMA average latency.** 70% local (90 ns), 30% remote (160 ns) ⇒ 0.7×90 + 0.3×160 = **111 ns**. Fixing placement to 95% local ⇒ 93.5 ns (−16%).

**N4. SMT throughput.** Thread alone: IPC 1.5 on a 6-wide core. With SMT, each thread gets IPC 1.0 ⇒ total 2.0 ⇒ +33% throughput, but each thread is **33% slower** — terrible for a latency-critical thread.

---

## 4. Interview Questions

### Basic
- **Q: Flynn's taxonomy?** (2.1.)
- **Q: What is SIMD? Give an example.** (2.2.)
- **Q: Multithreading vs multicore?** Threads share one core's resources vs separate cores with private pipelines/caches.

### Intermediate
- **Q: What is SMT and when does it help/hurt?** Helps when a single thread can't fill issue slots (memory-bound, low ILP); hurts per-thread latency and cache capacity.
- **Q: Why might the compiler not vectorise my loop?** (2.2 list.)
- **Q [QCOM][APPLE]: Why heterogeneous cores?** Perf/W; most work doesn't need a big core.
- **Q: What is NUMA and first-touch?** (2.5.)

### Advanced / killer follow-ups
- **Q [NVIDIA]: SIMD vs SIMT?** SIMD: explicit vector instructions, programmer/compiler manages width & masks. SIMT: scalar per-thread code; hardware groups threads into warps and handles divergence with masks/reconvergence; each thread has its own registers and can take its own branch (at a cost).
- **Q [HFT]: Would you enable hyper-threading on a trading server?** Usually no for the critical cores (or leave siblings idle): shared L1/L2/ports cause jitter; maybe yes for non-critical cores.
- **Q: SVE's "vector-length agnostic" — why?** One binary scales from 128- to 2048-bit hardware via predicate-driven loops (`whilelt`), no recompiling per width.
- **Q: Why can AVX-512 code sometimes make a server slower?** (Older Intel) frequency licence drop for heavy 512-bit instructions slows the whole core, and nearby scalar code — know as "AVX frequency offset".

---

## 5. Common Traps
- Assuming SIMD speeds up memory-bound loops proportionally.
- Assuming more threads = faster (contention, false sharing, SMT sharing, NUMA).
- Forgetting the scalar tail / alignment handling.

---

## 6. Revision Summary
- Flynn: SISD, SIMD, MISD, MIMD; GPU = SIMT.
- SIMD widths: SSE 128, AVX2 256, AVX-512 512 (masks), NEON 128, SVE scalable.
- Vectorisation blockers: aliasing, loop-carried deps, branches, AoS, FP reductions.
- Multithreading: coarse, fine (GPU), SMT (HT) — HFT disables on hot cores.
- Multicore: private L1/L2, shared LLC, ring/mesh; big.LITTLE.
- NUMA: local ~90 ns vs remote ~150 ns; first-touch; numactl.
- Peak FLOPs = cores × f × FMA units × lanes × 2.
