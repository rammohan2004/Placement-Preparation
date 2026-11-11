# 08. Main Memory — DRAM, HBM, GDDR, LPDDR

> **Asked at:** [NVIDIA] HBM vs GDDR, memory bandwidth math · [QCOM] [APPLE] LPDDR, unified memory, power · [TI] SRAM vs DRAM, refresh · [HFT] why DRAM latency ~ 80–100 ns and what row-buffer locality means.
> **Prerequisites:** [06 Cache](06-Cache-Memory.md).

---

## 1. Intuition
DRAM stores each bit as **charge on a tiny capacitor** (1 transistor + 1 capacitor) — dense and cheap, but slow, leaky (needs refresh) and read-destructive. It is organised like a **library of huge 2-D tables (banks)**: to read anything you first copy a whole **row** (~1–8 KB) into a **row buffer**, then read columns from it. Accesses to an already-open row are fast; switching rows is slow. Bandwidth has grown enormously; **latency has stayed ~flat (~15 ns core timings) for 20 years.**

---

## 2. Deep Dive

### 2.1 SRAM vs DRAM
| | SRAM | DRAM |
|---|---|---|
| Cell | 6 transistors (cross-coupled inverters) | 1 transistor + 1 capacitor |
| Density / cost | low / high | high / low |
| Speed | fast (ns, sub-ns) | slower (tens of ns) |
| Refresh | no (static while powered) | **yes**, every 64 ms (32 ms hot) |
| Read | non-destructive | destructive → sense amp restores |
| Use | registers, caches, scratchpads, GPU shared memory | main memory |

### 2.2 Organisation hierarchy
```
 Memory controller (on CPU die)
   └─ Channel (independent 64-bit bus; DDR5 = 2 × 32-bit sub-channels)
        └─ DIMM
             └─ Rank (set of chips that respond together, filling the bus width)
                  └─ Chip (x4/x8/x16 data pins)
                       └─ Bank (DDR4: 16 in 4 bank groups; DDR5: 32)
                            └─ 2-D array of rows × columns + ROW BUFFER (sense amps)
```

### 2.3 Access protocol and timings
```
 ACTIVATE (row r) ──tRCD──► READ/WRITE (column c) ──CL──► data burst ──► ... ──► PRECHARGE ──tRP──► (bank idle)
                       └──────────── tRAS (min row-open time) ────────────┘
```
| Timing | Meaning | Typical |
|---|---|---|
| **CL / tCAS** | column read → first data | ~14 ns |
| **tRCD** | activate → column command | ~14 ns |
| **tRP** | precharge (close row) | ~14 ns |
| tRAS | min time row stays open | ~32 ns |
| tRC = tRAS + tRP | min time between activates to the same bank | ~46 ns |
| tRFC | refresh command duration | ~300–400 ns |

DIMM labels like **DDR4-3200 16-18-18-38** = CL-tRCD-tRP-tRAS **in I/O clock cycles** (1600 MHz → 0.625 ns each).

**Three access cases:**
| Case | Latency (DRAM part) |
|---|---|
| **Row-buffer hit** (row already open) | CL |
| Row closed (bank precharged) | tRCD + CL |
| **Row conflict** (different row open) | tRP + tRCD + CL |
Plus controller queueing, on-chip network, cache misses through L1/L2/L3 → **~80–120 ns total load-to-use** on a typical server.

**Policies:** open-page (keep row open, bet on locality) vs closed-page (precharge immediately, bet on randomness). Scheduler **FR-FCFS** (first-ready, first-come-first-served) prioritises row hits.

### 2.4 Bandwidth
```
Peak BW = transfer rate (MT/s) × bus width (bytes) × #channels
```
"DDR" = data on both clock edges. A 64-byte cache line = burst of 8 × 8 bytes (DDR4) or 16 × 4 bytes per DDR5 sub-channel.

**Parallelism inside DRAM:** many banks can have operations in flight; address mapping interleaves consecutive lines across channels/banks so streaming uses them all (**bank-level parallelism**). Achievable BW is typically ~70–90% of peak.

### 2.5 Refresh
Each row must be refreshed every 64 ms. Controller issues REF every **tREFI = 7.8 µs**; during tRFC the rank (or bank, with per-bank refresh) is unavailable ⇒ few-% bandwidth loss and occasional latency spikes (an [HFT] tail-latency source). Higher-density chips ⇒ longer tRFC.

### 2.6 Memory technologies [NVIDIA][QCOM][APPLE]
| Tech | Where | Interface | Per-device BW (approx.) | Key idea |
|---|---|---|---|---|
| DDR4 / DDR5 | PCs, servers | 64-bit DIMM channel | DDR5-6400: 51.2 GB/s/channel | capacity, modularity |
| **LPDDR5/5X** | phones, Apple M-series, Snapdragon laptops | 16-bit channels, soldered/on-package | Apple M-series: 100–800 GB/s total | low power, short traces |
| **GDDR6/6X/7** | gaming GPUs | 32-bit per chip, very high per-pin rate (16–32+ Gbps) | 384-bit × 21 Gbps ≈ 1 TB/s (RTX 4090) | bandwidth/$ |
| **HBM2e/3/3e** | data-center GPUs/AI (NVIDIA A100/H100/B200, AMD MI300) | **1024-bit per stack**, 3-D stacked with TSVs on a silicon interposer | HBM3: ~819 GB/s/stack; H100 ≈ 3.35 TB/s; B200 ≈ 8 TB/s | massive width at modest clock ⇒ best BW/W |
- **Unified memory (Apple):** CPU, GPU, NPU share one LPDDR pool on-package — no PCIe copies.
- **ECC:** SEC-DED (Hamming + parity) on server DIMMs; DDR5 adds on-die ECC; HBM has ECC.
- **Row Hammer:** repeatedly activating a row flips bits in neighbouring rows — security issue; mitigations: TRR, higher refresh, ECC.

### 2.7 Implications for software
- Sequential access ⇒ row-buffer hits + prefetching ⇒ near-peak bandwidth.
- Random access ⇒ row conflicts + TLB misses ⇒ latency-bound; use more parallelism (independent misses) to hide it.
- **NUMA:** each socket has its own controllers; remote memory ~1.5–2× latency ([11](11-SIMD-Multithreading-Multicore.md)).
- On GPUs, **coalesced** accesses map to full bursts; scattered accesses waste bandwidth ([12](12-GPU-Architecture.md)).

---

## 3. Worked Numericals

**N1. DDR4-3200 dual-channel peak BW.** 3200 MT/s × 8 B × 2 = **51.2 GB/s**.

**N2. DDR4-3200 CL16 latency.** I/O clock = 1600 MHz ⇒ 0.625 ns. CL = 16 × 0.625 = **10 ns**. With tRCD = tRP = 18: row closed = (18 + 16) × 0.625 = 21.25 ns; row conflict = (18 + 18 + 16) × 0.625 = **32.5 ns** (+ 2.5 ns burst for 64 B).

**N3. Burst time for one cache line.** 64 B over a 64-bit bus = 8 transfers at 3200 MT/s = 8 / 3.2×10⁹ = **2.5 ns**.

**N4. HBM3 stack.** 1024 bits × 6.4 Gb/s per pin = 6553.6 Gb/s = **819.2 GB/s**. 5 stacks ≈ 4.1 TB/s.

**N5. GDDR6X card.** 384-bit × 21 Gb/s = 8064 Gb/s = **1008 GB/s**.

**N6. Refresh overhead.** tRFC = 350 ns, tREFI = 7.8 µs ⇒ 350 / 7800 ≈ **4.5%** of time the rank is busy refreshing.

**N7. Little's law check.** To sustain 51.2 GB/s with 90 ns latency: 51.2 GB/s × 90 ns ≈ **4.6 KB ≈ 72 cache lines in flight** — more than one core's ~12–16 miss buffers ⇒ a single thread cannot saturate DRAM bandwidth.

---

## 4. Interview Questions

### Basic
- **Q: SRAM vs DRAM?** (2.1.)
- **Q: Why does DRAM need refresh?** Capacitor charge leaks; must be read & rewritten every ~64 ms.
- **Q: What does "DDR" mean?** Transfers on both rising and falling clock edges.

### Intermediate
- **Q: Explain row-buffer hit, miss, conflict.** (2.3.)
- **Q: What do the numbers 16-18-18-38 mean?** CL-tRCD-tRP-tRAS in cycles.
- **Q: Why doesn't DRAM latency improve while bandwidth does?** Latency is dominated by analog sensing of tiny capacitors and long wordlines/bitlines (physics); bandwidth scales with parallelism (more pins, faster I/O, more banks/channels).
- **Q [QCOM][APPLE]: Why LPDDR in phones?** Lower voltage, low-power states, narrow channels, on-package placement ⇒ energy per bit much lower.

### Advanced / killer follow-ups
- **Q [NVIDIA]: HBM vs GDDR — trade-offs?** HBM: very wide (1024 bits/stack), lower clock, stacked with TSVs on interposer ⇒ highest BW and BW/W, but expensive, limited capacity, complex packaging (CoWoS). GDDR: narrow and very fast per pin, cheaper, on PCB, more power per bit.
- **Q: How does the memory controller map addresses to channels/banks and why does it matter?** Interleaving (often XOR-hashed bits) spreads consecutive lines over channels/banks to maximise parallelism and avoid pathological strides hitting one bank.
- **Q: What is Row Hammer?** (2.6.)
- **Q [HFT]: Why might a perfectly cache-resident program still see rare DRAM-like latency spikes?** Not DRAM-related only — but refresh, other cores' traffic, SMIs, frequency changes; plus occasional cache eviction by other processes on the shared L3 (noisy neighbours) — use cache allocation technology (Intel CAT) / core isolation.

---

## 5. Common Traps
- Calling DDR4-3200 a "3200 MHz" clock — it's 3200 MT/s, 1600 MHz I/O clock.
- Ignoring tRP/tRCD when computing a random access latency.
- Thinking more bandwidth = lower latency.

---

## 6. Revision Summary
- DRAM = 1T1C, refresh 64 ms, destructive read, row buffer.
- Hierarchy: channel → rank → chip → bank → row/col.
- Hit: CL · closed: tRCD + CL · conflict: tRP + tRCD + CL (~10–35 ns DRAM; ~80–120 ns end-to-end).
- BW = MT/s × width × channels. DDR4-3200 = 25.6 GB/s/channel.
- HBM = 1024-bit/stack stacked (TB/s), GDDR = fast narrow, LPDDR = low power, DDR = capacity.
- FR-FCFS; open vs closed page; bank-level parallelism; Row Hammer.
