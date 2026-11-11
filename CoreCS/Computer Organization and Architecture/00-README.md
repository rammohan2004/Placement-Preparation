# Computer Organization & Architecture — Index

> The single most important Core CS subject for NVIDIA, Apple, Qualcomm, TI and HFT interviews.
> Interviewers don't want definitions — they want you to **reason about performance**: *"Why is this loop slow?"*, *"What happens on a cache miss?"*, *"Why does adding a thread make it slower?"*

## Study order
| # | Note | Core idea | Time | Must for |
|---|---|---|---|---|
| 01 | [Number Representation & Arithmetic](01-Number-Representation-and-Arithmetic.md) | 2's complement, IEEE-754, adders, multipliers | 3 h | TI, QCOM, NVIDIA HW, HFT |
| 02 | [Performance Metrics & Laws](02-Performance-Metrics-and-Laws.md) | Iron law, CPI, Amdahl, roofline | 2 h | All |
| 03 | [ISA & Instruction Formats](03-ISA-and-Instruction-Formats.md) | RISC vs CISC, addressing, endianness, calling conventions | 2 h | QCOM, TI, APPLE |
| 04 | [Pipelining & Hazards](04-Pipelining-and-Hazards.md) | 5-stage pipeline, forwarding, stalls | 4 h | All HW, APPLE, NVIDIA |
| 05 | [Branch Prediction](05-Branch-Prediction.md) | 2-bit counters, gshare, BTB, TAGE | 2 h | APPLE, NVIDIA, HFT |
| 06 | [Cache Memory](06-Cache-Memory.md) | Mapping, AMAT, write policies, 3Cs | 5 h | **Everyone** |
| 07 | [Virtual Memory & TLB](07-Virtual-Memory-and-TLB.md) | Paging, TLB, VIPT, huge pages | 3 h | All |
| 08 | [Main Memory (DRAM, HBM)](08-Main-Memory-DRAM.md) | Banks, rows, refresh, bandwidth | 2 h | NVIDIA, QCOM, APPLE |
| 09 | [ILP, Superscalar & Out-of-Order](09-ILP-Superscalar-Out-of-Order.md) | Tomasulo, ROB, renaming | 4 h | APPLE, NVIDIA, QCOM |
| 10 | [Cache Coherence & Memory Consistency](10-Cache-Coherence-and-Memory-Consistency.md) | MESI, false sharing, TSO, fences | 4 h | HFT, NVIDIA, APPLE, DESHAW |
| 11 | [SIMD, Multithreading & Multicore](11-SIMD-Multithreading-Multicore.md) | Flynn, SMT, AVX, NUMA | 2 h | NVIDIA, HFT |
| 12 | [GPU Architecture](12-GPU-Architecture.md) | SIMT, warps, coalescing, occupancy | 4 h | **NVIDIA** |
| 13 | [I/O, Interrupts & DMA](13-IO-Interrupts-DMA.md) | MMIO, ISR, DMA, buses | 2 h | **TI, QCOM** |
| 14 | [Low-Latency Hardware Awareness](14-Low-Latency-Hardware-for-HFT.md) | Mechanical sympathy for trading systems | 3 h | **HFT, DESHAW** |
| 15 | [Interview Question Bank](15-Interview-Question-Bank.md) | Company-wise questions | ongoing | All |
| 16 | [Cheat Sheet](16-Cheat-Sheet.md) | Formulas & numbers on one page | revision | All |

**Total ≈ 45–50 hours** for a thorough first pass (≈ 2–3 weeks at 3 h/day alongside DSA).

## How to study each note
1. Read *Intuition* + *Deep Dive*.
2. Close the note, solve the *Worked Numericals* yourself, then compare.
3. Answer the *Interview Questions* **out loud** (interviews are spoken!).
4. Ask Claude: *"mock interview me on <topic>"* or *"give me 5 numericals on <topic>"*.
5. Update the progress table in the root [README](../../README.md).

## Recommended references
| Book / resource | Use for |
|---|---|
| Patterson & Hennessy — *Computer Organization and Design* (RISC-V ed.) | Fundamentals (ch. 1–5) |
| Hennessy & Patterson — *Computer Architecture: A Quantitative Approach* | OoO, coherence, GPUs, depth |
| Hamacher — *Computer Organization* | Arithmetic, I/O (GATE style) |
| Ulrich Drepper — *What Every Programmer Should Know About Memory* | Caches/DRAM for HFT |
| NVIDIA CUDA C++ Programming Guide & Best Practices Guide | GPU notes |
| Onur Mutlu's lectures (ETH Zürich / CMU, YouTube) | Best free lectures on all of this |
| Agner Fog's microarchitecture manual | x86 microarch details (HFT) |
