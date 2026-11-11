# 16. COA Cheat Sheet (read the night before)

## Formulas
| Topic | Formula |
|---|---|
| Iron law | T = IC × CPI × T_clk = IC × CPI / f |
| Weighted CPI | Σ fᵢ × CPIᵢ |
| CPI with stalls | CPI_ideal + mem-stalls/instr + branch-stalls/instr |
| Amdahl | S = 1 / ((1 − f) + f/s); max = 1/(1 − f) |
| Gustafson | S = (1 − f) + f·N |
| Little's law | in-flight = throughput × latency |
| Roofline | perf = min(peak FLOP/s, AI × BW); ridge = peak / BW |
| Dynamic power | P = αCV²f (≈ V³ with DVFS) |
| Pipeline time | (k + n − 1) × τ; τ = max stage + latch |
| Pipeline speedup | n·k / (k + n − 1) → k (ideal) |
| Branch CPI | branch freq × mispredict rate × penalty |
| Cache geometry | lines = C/B; sets = lines/ways; offset = log₂B; index = log₂sets; tag = A − index − offset |
| AMAT | HT₁ + MR₁ × (HT₂ + MR₂ × MP₂) |
| Global L2 MR | MR₁ × MR₂(local) |
| Mem stalls | accesses/instr × MR × MP |
| Page table (flat) | 2^(VA − offset) × PTE size |
| Levels | ⌈VPN bits / log₂(PTEs per page)⌉ |
| EAT (TLB) | h(t + m) + (1 − h)(t + (L + 1)m) |
| EAT (faults) | (1 − p)m + p × fault time |
| TLB reach | entries × page size |
| VIPT | cache size / ways ≤ page size |
| DRAM BW | MT/s × bus bytes × channels |
| DRAM latency | hit CL · closed tRCD + CL · conflict tRP + tRCD + CL |
| Peak FLOPs | cores × f × FMA units × lanes × 2 |
| GPU occupancy | min(regs, smem, threads, blocks limit) warps / max warps |
| Bank conflict degree | gcd(stride, 32) |
| Tiled GEMM AI (FP32) | T/4 FLOP/B |
| Independent chains needed | latency × throughput |

## Numbers
| | |
|---|---|
| n-bit 2's complement | −2ⁿ⁻¹ … 2ⁿ⁻¹ − 1 |
| float32 | 1/8/23, bias 127, ε = 2⁻²³ ≈ 1.19e−7, max ≈ 3.4e38, exact ints ≤ 2²⁴ |
| float64 | 1/11/52, bias 1023, ~16 digits, exact ints ≤ 2⁵³ |
| FP16 / BF16 | 1/5/10 (max 65504) / 1/8/7 (FP32 range) |
| L1 / L2 / L3 / DRAM | ~1 ns / ~4 ns / ~15 ns / ~90 ns |
| Mispredict | 15–20 cycles |
| Page sizes | 4 KB, 2 MB, 1 GB (Apple 16 KB) |
| x86-64 paging | 9+9+9+9+12 = 48 bits |
| DRAM refresh | 64 ms, tREFI 7.8 µs |
| DDR4-3200 | 25.6 GB/s per channel, CL16 = 10 ns |
| HBM3 stack | 1024 bits × 6.4 Gb/s ≈ 819 GB/s |
| PCIe 4 / 5 x16 | ~32 / ~64 GB/s per direction |
| Warp / banks | 32 threads / 32 banks × 4 B |
| H100 SM | 64K regs, ≤ 228 KB smem, ≤ 64 warps, 2048 threads |

## Key comparisons
| A | B |
|---|---|
| Write-through + no-allocate + write buffer | Write-back + write-allocate + dirty bit |
| Direct-mapped: fast hit, conflicts | Set-assoc: fewer conflicts, slower/power |
| Snooping: broadcast, small scale | Directory: home node, scalable |
| SC: one interleaving | TSO (x86): store→load reorder; ARM: weak |
| RISC: fixed length, load/store | CISC: variable, µops inside |
| Superscalar OoO: HW scheduling | VLIW: compiler scheduling (DSPs) |
| CPU: latency, OoO, big caches | GPU: throughput, SIMT, latency hiding |
| HBM: wide, stacked, TB/s | GDDR: narrow, fast pins; LPDDR: low power |
| Polling: low latency, wastes CPU | Interrupts: event-driven; DMA: bulk |
| UART async 2-wire | I²C 2-wire addressed / SPI 4-wire fast full-duplex |

## Hazards & fixes
| Problem | Fix |
|---|---|
| RAW | forwarding, stall, scheduling, OoO |
| Load-use | 1 bubble; compiler fills slot |
| WAR / WAW | renaming |
| Control | prediction (2-bit, gshare, TAGE), BTB, RAS |
| Structural | duplicate resources |
| Conflict misses | associativity, victim cache, padding |
| Capacity misses | bigger cache, tiling |
| Compulsory misses | bigger blocks, prefetching |
| False sharing | pad / align to 64–128 B |
| TLB misses | huge pages |
| DMA staleness | clean (TX) / invalidate (RX), aligned buffers |
| Warp divergence | warp-uniform branches |
| Uncoalesced | SoA, consecutive threads → consecutive addresses |
| Bank conflicts | pad [32][33] |

## C/C++ gotchas tied to COA
- Signed overflow = UB; `-1 < 0u` is false; `abs(INT_MIN)` overflows.
- `0.1 + 0.2 != 0.3`; FP not associative; NaN ≠ NaN.
- Struct padding: sort members large → small.
- `volatile` = MMIO/ISR visibility, **not** thread sync — use `std::atomic`.
- acquire/release for publish/subscribe; seq_cst store on x86 = `xchg`/`mfence`.
- Row-major arrays: iterate the last index innermost.
