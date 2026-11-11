# 15. COA Interview Question Bank — Company-wise

> These questions are **representative**, built from the topics each company is known to emphasise. They are not leaked papers. Practise answering **out loud in 1–2 minutes**, then expect a "why?" follow-up.
> Answer links point to the note section with the full explanation. Ask Claude: *"mock interview me with the NVIDIA questions from note 15"*.

---

## A. The "end-to-end" question (asked everywhere in some form)
**Q: Walk me through everything that happens when the CPU executes `x = a[i];` where `a[i]` is not in any cache.**
<details><summary>Model answer outline</summary>

1. **Fetch/decode:** instruction fetched from I-cache (ITLB translation), decoded to a load µop, renamed (destination gets a new physical register), placed in ROB, load queue, and scheduler. ([09](09-ILP-Superscalar-Out-of-Order.md))
2. **Address generation:** when `a` and `i` are ready, AGU computes `a + i*4`.
3. **Translation:** DTLB lookup in parallel with L1 set indexing (VIPT). On a TLB miss → STLB → hardware page walk (4 levels, may hit in caches). If the PTE is invalid → page fault → OS. ([07](07-Virtual-Memory-and-TLB.md))
4. **Memory disambiguation:** check store queue for older stores to the same address (store-to-load forwarding). ([09 §2.5](09-ILP-Superscalar-Out-of-Order.md))
5. **L1 miss** → allocate MSHR/fill buffer → L2 miss → L3 (shared, may consult snoop filter/directory for coherence — another core may hold it in M) → miss → memory controller → DRAM: row activate/column read/burst. ([06](06-Cache-Memory.md), [08](08-Main-Memory-DRAM.md), [10](10-Cache-Coherence-and-Memory-Consistency.md))
6. **Fill:** line returns (critical word first), installed in L3/L2/L1 per inclusion policy, possibly evicting dirty victims (write-back). Prefetchers may trigger on the pattern.
7. **Wakeup:** dependent instructions woken; meanwhile OoO executed independent younger instructions until ROB filled.
8. **Commit:** load retires in order from the ROB head; register mapping becomes architectural.
Total ≈ 80–120 ns if DRAM; ~1 ns if L1 hit.
</details>

---

## B. NVIDIA (Architecture / ASIC / CUDA / Perf)
1. Draw the 5-stage pipeline; show forwarding paths; why does load-use still stall? → [04 §2.6–2.7](04-Pipelining-and-Hazards.md)
2. Cache address breakdown for a given config; compute tag overhead. → [06 N1–N2](06-Cache-Memory.md)
3. Direct-mapped vs set-associative: hit time vs miss rate trade-off; when would you choose DM? → [06 §2.3](06-Cache-Memory.md)
4. What is a non-blocking cache? How many MSHRs do you need to sustain X GB/s? (Little's law) → [06 §4](06-Cache-Memory.md), [02 §2.5](02-Performance-Metrics-and-Laws.md)
5. Write-back vs write-through; write-allocate. Which for a GPU L2? (write-back) → [06 §2.6](06-Cache-Memory.md)
6. Explain MESI; why might GPUs not keep L1s coherent? → [10](10-Cache-Coherence-and-Memory-Consistency.md), [12 §2.5](12-GPU-Architecture.md)
7. CPU vs GPU design philosophy; SIMD vs SIMT. → [12 §1](12-GPU-Architecture.md), [11 §4](11-SIMD-Multithreading-Multicore.md)
8. Warp divergence: causes, cost, mitigation. → [12 §2.4](12-GPU-Architecture.md)
9. Memory coalescing with stride examples; shared-memory bank conflicts with padding. → [12 §2.6–2.7](12-GPU-Architecture.md)
10. Compute occupancy for a kernel config; is higher occupancy always better? → [12 N1, §2.8](12-GPU-Architecture.md)
11. Roofline: is GEMV / GEMM / softmax memory- or compute-bound? → [02 §2.6](02-Performance-Metrics-and-Laws.md)
12. HBM vs GDDR trade-offs. → [08 §2.6](08-Main-Memory-DRAM.md)
13. FP16 vs BF16 vs FP8; why tensor cores accumulate in FP32. → [01 §2.5](01-Number-Representation-and-Arithmetic.md)
14. Design a branch predictor for a small embedded core vs a big core. → [05](05-Branch-Prediction.md)
15. Register renaming: which hazards does it remove? How many physical registers needed? → [09 N4](09-ILP-Superscalar-Out-of-Order.md)
16. Why is a 64-bit adder not a ripple-carry adder? Explain carry-lookahead / Kogge-Stone. → [01 §2.6](01-Number-Representation-and-Arithmetic.md)
17. Optimise a matrix transpose kernel. → [12 §2.6](12-GPU-Architecture.md)

## C. Texas Instruments (Embedded / Digital / Analog-mixed roles)
1. 2's complement range; detect overflow in hardware and in C. → [01 §2.2](01-Number-Representation-and-Arithmetic.md)
2. Q15 multiply with rounding and saturation. → [01 §2.3](01-Number-Representation-and-Arithmetic.md)
3. Big vs little endian; write detection code; when does it matter? → [03 §2.6](03-ISA-and-Instruction-Formats.md)
4. Struct padding and alignment; reorder to minimise size. → [03 §2.7](03-ISA-and-Instruction-Formats.md)
5. `volatile` — what and why; is it enough for ISR sharing? → [13 §2.1, 2.4](13-IO-Interrupts-DMA.md)
6. Polling vs interrupts vs DMA. → [13 §2.2](13-IO-Interrupts-DMA.md)
7. Interrupt latency and what affects it; nested interrupts; NVIC tail-chaining. → [13 §2.3–2.4](13-IO-Interrupts-DMA.md)
8. DMA + cache coherence problem and fix. → [13 §2.6](13-IO-Interrupts-DMA.md)
9. UART vs SPI vs I²C; why pull-ups on I²C? → [13 §2.7](13-IO-Interrupts-DMA.md)
10. Pipeline speedup numerical with unequal stages. → [04 N1](04-Pipelining-and-Hazards.md)
11. SRAM vs DRAM; why DRAM needs refresh. → [08 §2.1](08-Main-Memory-DRAM.md)
12. Harvard vs von Neumann (DSPs use modified Harvard: separate I/D buses for single-cycle MAC with fetch). → see answer below
13. VLIW in C6000 DSPs; software pipelining. → [09 §2.2](09-ILP-Superscalar-Out-of-Order.md)
14. Circular buffers and bit-reversed addressing. → [03 §2.5](03-ISA-and-Instruction-Formats.md)
15. Booth multiplication by hand. → [01 N3](01-Number-Representation-and-Arithmetic.md)

<details><summary>Q12 answer: Harvard vs von Neumann</summary>

**Von Neumann:** single memory and bus for instructions and data ⇒ structural bottleneck (can't fetch an instruction and data in the same cycle). **Harvard:** separate instruction and data memories/buses ⇒ simultaneous access. **Modified Harvard** (almost all modern CPUs/DSPs): separate L1 I-cache and D-cache (Harvard at the core), unified memory behind them (von Neumann for the programmer). DSPs often have *multiple* data buses so a MAC can fetch two operands + an instruction per cycle.
</details>

## D. Qualcomm (CPU/GPU/DSP/modem SW and HW)
1. ARM vs x86; RISC vs CISC today. → [03 §2.3](03-ISA-and-Instruction-Formats.md)
2. Thumb, AArch64 register conventions, conditional execution removal. → [03 §2.9](03-ISA-and-Instruction-Formats.md)
3. Pipeline hazards and their solutions; CPI with stalls numerical. → [04](04-Pipelining-and-Hazards.md)
4. Cache mapping and replacement; AMAT with L1/L2. → [06](06-Cache-Memory.md)
5. VIPT cache constraint. → [07 §2.6](07-Virtual-Memory-and-TLB.md)
6. TLB and multi-level page tables; page table size numericals. → [07](07-Virtual-Memory-and-TLB.md)
7. ARM weak memory model; DMB vs DSB vs ISB. → [10 §3.3](10-Cache-Coherence-and-Memory-Consistency.md), answer below
8. big.LITTLE and DVFS; P ∝ CV²f. → [11 §2.4](11-SIMD-Multithreading-Multicore.md), [02 §2.7](02-Performance-Metrics-and-Laws.md)
9. Hexagon DSP as VLIW + hardware multithreading. → [09](09-ILP-Superscalar-Out-of-Order.md)
10. LPDDR vs DDR; why mobile uses LPDDR. → [08](08-Main-Memory-DRAM.md)
11. Bit manipulation: count set bits, power of 2, reverse bits. → [01 §2.9](01-Number-Representation-and-Arithmetic.md)
12. AXI vs AHB vs APB. → [13 §2.7](13-IO-Interrupts-DMA.md)

<details><summary>Q7 answer: DMB vs DSB vs ISB</summary>

- **DMB** (data memory barrier): orders memory accesses before vs after it (as observed by others); execution continues.
- **DSB** (data synchronization barrier): stronger — no instruction after it executes until all prior memory accesses (and cache/TLB maintenance) **complete**. Used before WFI, after TLB/cache maintenance, before ringing a device doorbell where completion matters.
- **ISB** (instruction synchronization barrier): flushes the pipeline so later instructions are re-fetched — needed after changing system registers (MMU config, VBAR) or self-modifying code.
</details>

## E. Apple (CPU/GPU architecture, SoC, performance)
1. How does an out-of-order core work end-to-end (rename, ROB, IQ, LSQ, commit)? → [09 §2.4](09-ILP-Superscalar-Out-of-Order.md)
2. Why can Apple cores decode 8+ instructions/cycle? → [03 §4](03-ISA-and-Instruction-Formats.md)
3. Why is the M-series L1 so large? (16 KB pages, VIPT) → [07 §2.6](07-Virtual-Memory-and-TLB.md)
4. TAGE predictor; RAS; indirect prediction. → [05](05-Branch-Prediction.md)
5. Memory disambiguation and store-to-load forwarding. → [09 §2.5](09-ILP-Superscalar-Out-of-Order.md)
6. TSO vs weak ordering; how Rosetta 2 handles it. → [10 §3.3](10-Cache-Coherence-and-Memory-Consistency.md)
7. Unified memory architecture: pros/cons. → [08 §2.6](08-Main-Memory-DRAM.md)
8. Perf/W: why wide-and-slow beats narrow-and-fast. → [02 §2.7](02-Performance-Metrics-and-Laws.md)
9. Precise exceptions in an OoO machine. → [04 §2.9](04-Pipelining-and-Hazards.md), [09](09-ILP-Superscalar-Out-of-Order.md)
10. Spectre: what and why speculation leaks. → [05 §2.9](05-Branch-Prediction.md)

## F. Amazon (SDE — COA as concept checks)
1. What is a cache; why does it help? Locality. → [06 §1](06-Cache-Memory.md)
2. Process vs thread memory; virtual memory benefits. → [07 §1](07-Virtual-Memory-and-TLB.md)
3. What is a page fault? Thrashing? → [07 §2.7, 2.9](07-Virtual-Memory-and-TLB.md)
4. Latency vs throughput; Amdahl's law. → [02](02-Performance-Metrics-and-Laws.md)
5. Why might a multithreaded program be slower than single-threaded? (contention, false sharing, overhead) → [10](10-Cache-Coherence-and-Memory-Consistency.md)
6. Endianness in network protocols. → [03 §2.6](03-ISA-and-Instruction-Formats.md)

## G. HFT — IMC, Optiver, Tower, Graviton, Quadeye
1. Latency numbers every programmer should know. → [14 §2.1](14-Low-Latency-Hardware-for-HFT.md)
2. Why is iterating a 2-D array column-wise slow? Quantify. → [06 N6](06-Cache-Memory.md)
3. False sharing — explain, detect, fix. → [10 §2.4](10-Cache-Coherence-and-Memory-Consistency.md)
4. Store-buffer litmus test on x86; how to fix with fences / `seq_cst`. → [10 §3.2](10-Cache-Coherence-and-Memory-Consistency.md)
5. Implement a lock-free SPSC queue and justify every memory order. → [10 §3.5](10-Cache-Coherence-and-Memory-Consistency.md)
6. Sorted vs unsorted array branch question; branchless alternative. → [05 §2.8](05-Branch-Prediction.md)
7. Why are huge pages useful? Downsides of THP? → [07 §2.8](07-Virtual-Memory-and-TLB.md)
8. What happens on a context switch, and why does it hurt latency beyond the switch itself? (cache/TLB pollution) → [14](14-Low-Latency-Hardware-for-HFT.md)
9. SMT on or off for a trading box? → [11 §2.3](11-SIMD-Multithreading-Multicore.md)
10. Design an order book data structure. → [14 §2.4, §4](14-Low-Latency-Hardware-for-HFT.md)
11. Cost of virtual functions; CRTP. → [14 §2.4](14-Low-Latency-Hardware-for-HFT.md)
12. Why are floats bad for prices? → [01 §4](01-Number-Representation-and-Arithmetic.md)
13. Measure p99 latency correctly; benchmarking pitfalls. → [14 §2.5](14-Low-Latency-Hardware-for-HFT.md)
14. Polling vs interrupts for NIC; kernel bypass. → [13 §2.8](13-IO-Interrupts-DMA.md)
15. Summation loop at 1/8 peak — why? → [09 §2.8](09-ILP-Superscalar-Out-of-Order.md)

## H. D. E. Shaw
1. Signed vs unsigned comparison trap (`-1 < 0u`). → [01 §5](01-Number-Representation-and-Arithmetic.md)
2. `sizeof` a struct with padding. → [03 N4](03-ISA-and-Instruction-Formats.md)
3. Cache hit/miss trace for DM / 2-way / FA. → [06 N3](06-Cache-Memory.md)
4. AMAT and CPI-with-stalls numericals. → [06 N4–N5](06-Cache-Memory.md)
5. Page table size and EAT with TLB. → [07 N1–N3](07-Virtual-Memory-and-TLB.md)
6. Pipeline: count stalls for a code snippet. → [04 N4–N5](04-Pipelining-and-Hazards.md)
7. Amdahl puzzle: which enhancement is better? → [02 N4](02-Performance-Metrics-and-Laws.md)
8. Is `volatile` enough for thread synchronisation? → [10 §5](10-Cache-Coherence-and-Memory-Consistency.md)
9. 2-bit predictor accuracy on alternating pattern. → [05 N2](05-Branch-Prediction.md)
10. Why is `i++` on a shared int not thread-safe even on a single-core machine? (load-modify-store can be interrupted between steps)

---

## I. Rapid-fire (answer in one line each)
| # | Question | Answer |
|---|---|---|
| 1 | Range of 16-bit signed int | −32768 … 32767 |
| 2 | float32 exponent bias | 127 |
| 3 | Largest exact contiguous integer in double | 2⁵³ |
| 4 | Typical cache line size | 64 B (Apple M: 128 B) |
| 5 | Index bits for 32 KB 8-way 64 B cache | 6 (64 sets) |
| 6 | Which hazard can't forwarding fix completely? | Load-use |
| 7 | WAR/WAW removed by? | Register renaming |
| 8 | MESI state allowing silent write | E (and M) |
| 9 | x86 memory model | TSO (store→load reordering) |
| 10 | 2-bit counter loop mispredicts per loop execution | 1 |
| 11 | Structure for return prediction | Return Address Stack |
| 12 | Warp size (NVIDIA) | 32 |
| 13 | Shared-memory banks | 32 × 4 B |
| 14 | VIPT constraint | cache size / ways ≤ page size |
| 15 | x86-64 page-table levels | 4 (5 with LA57) |
| 16 | DRAM refresh period | 64 ms |
| 17 | DDR4-3200 channel bandwidth | 25.6 GB/s |
| 18 | HBM interface width per stack | 1024 bits |
| 19 | Amdahl max speedup with 5% serial | 20× |
| 20 | Dynamic power formula | αCV²f |
