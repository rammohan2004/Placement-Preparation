# 02. Performance Metrics & Laws

> **Asked at:** Everyone. [NVIDIA] roofline, Little's law, bandwidth-bound vs compute-bound · [APPLE] [QCOM] performance-per-watt · [HFT] latency vs throughput, tail latency · [AMZN] [DESHAW] Amdahl numericals.
> **Prerequisites:** none.

---

## 1. Intuition
There are only three knobs on execution time: **how many instructions** you run, **how many cycles each takes**, and **how long a cycle is**. Every architecture idea (pipelining, caches, OoO, SIMD, GPUs) is an attempt to shrink one knob without inflating another. And every speedup is capped by the part you *didn't* speed up (Amdahl).

---

## 2. Deep Dive

### 2.1 The Iron Law of Performance
```
CPU time = Instruction Count × CPI × Clock period
         = IC × CPI / f
```
| Knob | Affected by |
|---|---|
| IC (instructions) | Algorithm, compiler, ISA (CISC fewer instrs, SIMD fewer instrs) |
| CPI (cycles/instr) | Microarchitecture: pipelining, caches, branch prediction, OoO |
| Clock period | Circuit technology, pipeline depth, power limits |

- **IPC = 1/CPI.** Modern big cores reach IPC 3–6 on friendly code; memory-bound code can be < 0.5.
- **Weighted CPI** for an instruction mix: `CPI = Σ (fraction_i × CPI_i)`.
- **CPI with stalls:** `CPI = CPI_ideal + memory-stall cycles/instr + branch-stall cycles/instr`.

**Why MIPS / GHz are misleading:** MIPS = f / (CPI × 10⁶) ignores IC — a CISC chip with fewer, slower instructions can have lower MIPS but finish first. A deeper pipeline raises GHz but raises branch-misprediction penalty.

### 2.2 Speedup and Amdahl's Law
`Speedup = Time_old / Time_new`

If a fraction **f** of the execution time is sped up by factor **s**:
```
Speedup = 1 / ((1 − f) + f / s)          lim s→∞ : 1 / (1 − f)
```
- 95% parallel ⇒ never more than 20× no matter how many cores.
- Lesson: *make the common case fast*, and serial bottlenecks dominate at scale.

### 2.3 Gustafson's Law (the optimist's view)
Amdahl fixes the problem size. In reality, with more processors we solve **bigger problems** in the same time (weak scaling):
```
Scaled speedup = (1 − f) + f × N         (f = parallel fraction measured on the N-processor run)
```
GPUs and HPC live by Gustafson: more cores ⇒ bigger batch / mesh / model.

### 2.4 Latency vs Throughput
| | Latency | Throughput (bandwidth) |
|---|---|---|
| Definition | time for **one** task | tasks per unit time |
| Improved by | faster circuits, shorter paths, caches | parallelism, pipelining |
| Who cares most | **HFT** (tick-to-trade), interactive | **GPU**, servers, batch |
| Note | bandwidth is easy to buy (more lanes); latency is hard (physics) | |

*"Bandwidth improves with the square of latency improvement"* (Patterson) — historically bandwidth grew far faster than latency fell.

**Tail latency [HFT][AMZN]:** the p99 / p99.9 matters more than average. Sources: cache misses, page faults, interrupts, GC, context switches, SMIs, frequency changes.

### 2.5 Little's Law — the most underrated formula [NVIDIA][HFT]
```
Concurrency (items in flight) = Throughput × Latency
```
To keep a memory system busy you need **bytes-in-flight = bandwidth × latency**.
- One CPU core has only ~10–16 L1 miss buffers (line-fill buffers / MSHRs) ⇒ caps single-thread bandwidth.
- GPUs hide ~400–800 cycle memory latency by keeping **thousands of threads** in flight — that's Little's law in silicon.

### 2.6 The Roofline Model [NVIDIA]
```
Arithmetic Intensity (AI) = FLOPs performed / Bytes moved from DRAM
Attainable FLOP/s = min( Peak FLOP/s ,  AI × Peak Bandwidth )
Ridge point AI* = Peak FLOP/s / Peak BW
```
```
 FLOP/s
   │            _______________________  ← compute roof (peak)
   │          /
   │        /   ← memory roof (slope = BW)
   │      /
   │    /
   └──────┼─────────────────────────── AI (FLOP/byte)
         AI*  (left: memory-bound, right: compute-bound)
```
- Vector add, SAXPY, most element-wise ops → AI < 1 → **memory-bound**.
- Large matrix multiply → AI grows with tile size (O(n) per element) → **compute-bound**.
- Optimisation strategy: if memory-bound, *raise AI* (tiling, fusion, reuse in shared memory/cache) or reduce bytes (lower precision); if compute-bound, use tensor cores/SIMD/FMA.

### 2.7 Power and energy [APPLE][QCOM][TI]
```
P_dynamic = α · C · V² · f        (α = activity factor)
P_static  = V · I_leak
Energy    = Power × Time
```
- Since f roughly scales with V, **P ∝ V³** near the top of the DVFS curve ⇒ slightly lower clocks save a lot of power.
- **Dennard scaling ended (~2005):** voltage stopped scaling, so power density rose ⇒ "power wall" ⇒ multicore, specialised accelerators, **dark silicon**.
- Mobile metric = **performance per watt** and **energy per task** ("race to idle": finish fast, then sleep).
- Techniques: DVFS, clock gating (cuts α), power gating (cuts leakage), big.LITTLE heterogeneous cores.

### 2.8 Summarising benchmarks
- Times: arithmetic mean. Rates: harmonic mean. **Normalized ratios (SPEC): geometric mean** (independent of reference machine).

---

## 3. Worked Numericals

**N1. Weighted CPI.** 10⁹ instructions at 2 GHz. Mix: ALU 50% (CPI 1), load 20% (5), store 10% (3), branch 20% (2).
CPI = 0.5×1 + 0.2×5 + 0.1×3 + 0.2×2 = 0.5 + 1.0 + 0.3 + 0.4 = **2.2**
Time = 10⁹ × 2.2 × 0.5 ns = **1.1 s**

**N2. Compiler trade-off.** An optimisation cuts IC by 20% but raises CPI by 10%.
New time / old = 0.8 × 1.1 = 0.88 ⇒ speedup **1.136×** (worth it).

**N3. Amdahl.** 80% parallelisable, 8 cores: 1/(0.2 + 0.8/8) = 1/0.3 = **3.33×**. Infinite cores: **5×**.

**N4. Which enhancement? (H&P classic)** FP square root is 20% of time and can be made 10× faster; *or* all FP instructions (50% of time) made 1.6× faster.
- Sqrt: 1/(0.8 + 0.02) = **1.22×**
- All FP: 1/(0.5 + 0.3125) = **1.23×** → the broader, modest improvement wins slightly.

**N5. Roofline.** GPU: 100 TFLOP/s FP32, 2 TB/s HBM. Ridge AI* = 50 FLOP/B.
SAXPY `y[i] = a*x[i] + y[i]` (float): 2 FLOPs; bytes = 4 (x) + 4 (y read) + 4 (y write) = 12 ⇒ AI = 1/6.
Attainable = (1/6) × 2 TB/s ≈ **0.33 TFLOP/s = 0.33% of peak** → memory-bound; buying more FLOPs is useless.

**N6. Little's law.** DRAM latency 80 ns; a core has 12 miss buffers of 64 B.
Max single-core BW = 12 × 64 B / 80 ns = 768 B / 80 ns = **9.6 GB/s**, even if DRAM delivers 50+ GB/s. To saturate 50 GB/s you need 50 GB/s × 80 ns = **4000 B ≈ 63 lines in flight** → multiple cores or prefetching.

**N7. DVFS.** Lower V and f each by 15%: P_dyn ∝ V²f ⇒ 0.85³ ≈ **0.61** (39% power saved) for a 15% throughput loss.

**N8. Gustafson.** On 64 cores a run spends 5% in serial code: scaled speedup = 0.05 + 0.95×64 = **60.85×**.

---

## 4. Interview Questions

### Basic
- **Q: State the iron law and what affects each term.** (See 2.1.)
- **Q: Why is clock frequency alone a bad metric?** Ignores CPI and IC; deeper pipelines raise f but raise penalties.
- **Q: State Amdahl's law; what's the max speedup with 10% serial code?** 10×.

### Intermediate
- **Q: Latency vs throughput — give an example where improving one hurts the other.** Batching (Nagle, GPU batch size) raises throughput, raises latency. Deeper pipelines raise throughput, raise per-instruction latency.
- **Q [NVIDIA]: What is the roofline model? Is matrix-vector multiply compute or memory bound?** GEMV: each matrix element used once → AI ≈ 2 FLOPs / 4 B = 0.5 → memory-bound. GEMM is compute-bound for large tiles.
- **Q: Why did the industry move to multicore?** End of Dennard scaling ⇒ power wall; ILP wall (diminishing returns); memory wall.
- **Q [APPLE][QCOM]: Why does lowering frequency 10% save more than 10% power?** Voltage can drop too; P ∝ V²f.

### Advanced / killer follow-ups
- **Q [NVIDIA]: Using Little's law, explain why GPUs need so many threads.** Bytes in flight = BW × latency, e.g. 3 TB/s × 600 ns ≈ 1.8 MB in flight across the chip — only possible with tens of thousands of outstanding loads.
- **Q [HFT]: Your average latency is 2 µs but p99 is 50 µs. Hypotheses?** Cache/TLB misses after idle (cold cache), page faults, interrupts on the core, context switches, frequency scaling/C-states, SMI, GC/allocations, false sharing, logging. Fixes: core isolation, busy-polling, huge pages, pre-touching memory, cache warming.
- **Q: Why geometric mean for SPEC ratios?** Ratio of geometric means = geometric mean of ratios ⇒ result independent of the reference machine.
- **Q: Amdahl says 20× max, yet people see super-linear speedup. How?** More cores ⇒ more aggregate cache; the working set suddenly fits ⇒ CPI drops.

---

## 5. Common Traps
- Applying the speedup to the wrong fraction (f must be fraction of **time**, not of instructions).
- Comparing MIPS across ISAs.
- Forgetting that memory-bound code doesn't speed up with more FLOPs/cores sharing the same bandwidth.
- Averaging ratios arithmetically.

---

## 6. Revision Summary
| Formula | |
|---|---|
| Iron law | T = IC × CPI / f |
| Amdahl | 1 / ((1−f) + f/s); max 1/(1−f) |
| Gustafson | (1−f) + fN |
| Little | in-flight = throughput × latency |
| Roofline | min(peak, AI × BW); ridge = peak/BW |
| Dynamic power | αCV²f (≈ V³) |
| SPEC | geometric mean |
