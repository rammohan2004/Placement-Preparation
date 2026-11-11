# 12. GPU Architecture ⭐ (NVIDIA must-know)

> **Asked at:** **[NVIDIA]** (all roles: arch, ASIC, CUDA/SW, perf) — SIMT, warps, divergence, coalescing, shared-memory bank conflicts, occupancy, tensor cores, roofline · [QCOM] Adreno / [APPLE] Apple GPU basics · [HFT] GPU vs FPGA vs CPU for latency.
> **Prerequisites:** [02 Roofline/Little](02-Performance-Metrics-and-Laws.md), [06 Cache](06-Cache-Memory.md), [08 DRAM/HBM](08-Main-Memory-DRAM.md), [11 SIMD](11-SIMD-Multithreading-Multicore.md).

---

## 1. Intuition
A CPU is a few very smart workers (big OoO cores, huge caches, branch predictors) built to finish **one task fast** (latency). A GPU is **thousands of simple workers** built to finish **many tasks per second** (throughput). Instead of avoiding memory latency with caches and speculation, a GPU **hides** it: when one group of threads waits for memory, the scheduler instantly switches to another ready group — at zero cost, because every group's registers stay resident on-chip.

```
 CPU die area: big control + big caches, few ALUs     GPU die area: mostly ALUs + register files
 ┌─────────┬──────────┐                               ┌──┬──┬──┬──┬──┬──┬──┬──┐
 │ Control │ ALU ALU  │                               │SM│SM│SM│SM│SM│SM│SM│SM│
 │ (OoO,BP)│ ALU ALU  │                               ├──┼──┼──┼──┼──┼──┼──┼──┤
 ├─────────┴──────────┤                               │SM│SM│SM│SM│SM│SM│SM│SM│
 │   Large L2 / L3    │                               ├──┴──┴──┴──┴──┴──┴──┴──┤
 └────────────────────┘                               │ L2 cache  │ HBM ctrls │
                                                      └───────────────────────┘
```

---

## 2. Deep Dive

### 2.1 Hardware hierarchy (NVIDIA terms; numbers ≈ H100 SXM, for orientation)
| Level | Contents |
|---|---|
| GPU | 132 SMs, ~50 MB L2, 80 GB HBM3 @ ~3.35 TB/s, NVLink 4 (900 GB/s), PCIe 5 |
| **SM (Streaming Multiprocessor)** | 4 sub-partitions (SMSPs); 64K × 32-bit registers (256 KB); 256 KB combined L1 / shared memory (up to 228 KB shared); 4 tensor cores; 128 FP32 lanes; max 2048 threads (64 warps), max 32 blocks |
| SM sub-partition | 1 **warp scheduler** (issues 1 warp-instruction/cycle), register file slice, 32 FP32 lanes, 1 tensor core, LD/ST units, SFUs |
| Warp | **32 threads** executing in lockstep (SIMT) |
(AMD calls them CUs and **wavefronts** of 64 (or 32 in RDNA); Qualcomm Adreno and Apple GPUs follow similar ideas.)

### 2.2 CUDA programming model → hardware mapping
```
 Grid (kernel launch)          → whole GPU
   └ Thread Block (CTA, ≤1024 threads) → runs entirely on ONE SM; can share __shared__ memory & __syncthreads()
       └ Warp (32 consecutive threads)  → scheduling unit on an SM sub-partition
           └ Thread                    → one SIMT lane, private registers
 (Hopper adds Thread Block Clusters → distributed shared memory across SMs in a GPC)
```
```cuda
__global__ void saxpy(int n, float a, const float* x, float* y) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) y[i] = a * x[i] + y[i];
}
saxpy<<<(n + 255) / 256, 256>>>(n, 2.0f, d_x, d_y);
```
Blocks are independent and can run in any order ⇒ the same code scales from small to huge GPUs.

### 2.3 SIMT execution & latency hiding
- Each cycle, each warp scheduler picks a **ready** warp (operands available, no pending dependency) and issues one instruction for all 32 threads.
- Warps waiting on memory (~400–800 cycles for HBM) simply aren't selected. **Zero-cost switching** — registers of all resident warps live in the huge register file.
- Little's law ([02](02-Performance-Metrics-and-Laws.md)): enough warps × independent instructions must be in flight to cover latency. Two sources: **TLP** (more warps = occupancy) and **ILP** (independent instructions within a thread).
- GPUs have no branch predictors, no OoO execution, small caches per thread — the area goes into ALUs and registers.

### 2.4 Warp divergence
If threads in one warp take different branch paths, the warp executes **each path serially** with inactive lanes masked off, then **reconverges**.
```cuda
if (threadIdx.x % 2 == 0) A(); else B();          // every warp diverges → ~50% efficiency
if ((threadIdx.x / 32) % 2 == 0) A(); else B();   // warp-uniform → no divergence
```
- Volta+ **Independent Thread Scheduling**: per-thread PC & stack ⇒ divergent threads can interleave (enables intra-warp locks), but performance still requires convergence. Use `__syncwarp()` where warp-synchronous code is assumed.
- Mitigate: make branches warp-uniform, sort/bin work by type, use predication (`?:`) for short paths.

### 2.5 Memory hierarchy
| Memory | Scope | Location | Latency (approx.) | Notes |
|---|---|---|---|---|
| Registers | thread | SM register file | ~0 | 255 max/thread; excess **spills** to "local" memory (slow!) |
| **Shared memory** | block | on-SM SRAM (shares with L1) | ~20–30 cycles | programmer-managed scratchpad, 32 banks |
| L1 / texture cache | SM | on-SM | ~30 cycles | not coherent across SMs |
| L2 | device | on-chip | ~200 cycles | coherence point; atomics resolve here |
| Global (HBM/GDDR) | device | off-chip | ~400–800 cycles | TB/s bandwidth |
| Constant | device (cached) | | fast if all threads read **same** address (broadcast) | |
| Local | thread | actually global memory | slow | spills, large private arrays |

### 2.6 Memory coalescing ⭐
Global memory is accessed in **32-byte sectors** (and 128-byte cache lines = 4 sectors). The load/store unit combines a warp's 32 addresses into as few sector transactions as possible.
```
 Coalesced:  thread k reads a[base + k] (float)  → 32 × 4 B = 128 B = 4 sectors  ✅ 100% efficiency
 Stride 2:   a[base + 2k] → spans 256 B = 8 sectors, uses 128 B                   → 50%
 Stride 32:  a[base + 32k] → 32 different sectors, uses 4 B of each 32 B          → 12.5%
 Random:     up to 32 sectors per warp
```
Rules: consecutive threads → consecutive addresses; use **SoA**, not AoS; align base to 128 B; use vectorised loads (`float4`) when each thread needs several elements.
Matrix transpose: read coalesced rows into shared memory, write coalesced from shared memory (the canonical exercise).

### 2.7 Shared memory bank conflicts ⭐
Shared memory has **32 banks**, each 4 bytes wide; successive 4-byte words go to successive banks: `bank = (byte_address / 4) % 32`.
- All 32 threads hitting **different banks** (or the **same word** → broadcast) ⇒ 1 cycle.
- n threads hitting **different addresses in the same bank** ⇒ **n-way conflict** ⇒ serialized n times.
```cuda
__shared__ float tile[32][32];
float v = tile[threadIdx.x][0];      // column access: addresses 32 words apart → all bank 0 → 32-way conflict ❌
__shared__ float tile[32][33];       // pad by one column
float v = tile[threadIdx.x][0];      // address = tid*33 → bank = tid*33 % 32 = tid → conflict-free ✅
```

### 2.8 Occupancy
**Occupancy = active warps per SM / max warps per SM (64 on H100).** Limited by whichever resource runs out first:
1. Registers per thread × threads per block (64K registers per SM).
2. Shared memory per block.
3. Max threads per SM (2048) and max blocks per SM (32).
Higher occupancy ⇒ more latency hiding — but **not always faster**: kernels with high ILP and heavy register/shared-memory tiling (e.g., GEMM) often run best at low occupancy. Use the CUDA occupancy calculator / Nsight Compute.

### 2.9 Tensor cores & mixed precision
Dedicated units computing a **small matrix multiply-accumulate (D = A×B + C)** per instruction (warp-level `mma`; Hopper adds warpgroup-level `wgmma` and the **TMA** async copy engine). Inputs FP16/BF16/TF32/FP8/INT8, accumulate in FP32 (or FP16). They provide the bulk of AI FLOPs (an order of magnitude more than FP32 CUDA cores). Formats: see [01](01-Number-Representation-and-Arithmetic.md).

### 2.10 Synchronisation & communication
- `__syncthreads()` — barrier for a block (+ shared-memory visibility). Must be reached by **all** threads of the block (never inside divergent code).
- Warp-level primitives: `__shfl_sync`, `__ballot_sync`, `__reduce_add_sync` — register-to-register exchange within a warp, no shared memory.
- Atomics on global (resolved in L2) and shared memory; heavy contention serialises.
- No grid-wide barrier except kernel boundaries or cooperative groups (`grid.sync()` with cooperative launch).

### 2.11 Host ↔ device
- PCIe 4.0 x16 ≈ 32 GB/s/direction, PCIe 5.0 x16 ≈ 64 GB/s — **~50–100× less than HBM** ⇒ minimise transfers, keep data resident.
- **Pinned (page-locked) host memory** enables DMA and async copies.
- **Streams** overlap copy (copy engines) with compute; CUDA Graphs reduce launch overhead (~µs per launch).
- NVLink / NVSwitch for GPU-GPU; GPUDirect RDMA / Storage for NIC/SSD → GPU without CPU bounce.

### 2.12 Classic kernels to know
**(a) Tiled matrix multiply (shared memory):**
```cuda
#define T 32
__global__ void matmul(const float* A, const float* B, float* C, int N) {
    __shared__ float As[T][T], Bs[T][T];   // As[ty][k]: broadcast within a row; Bs[k][tx]: consecutive banks → no conflicts
    int r = blockIdx.y * T + threadIdx.y, c = blockIdx.x * T + threadIdx.x;
    float acc = 0.f;
    for (int k0 = 0; k0 < N; k0 += T) {
        As[threadIdx.y][threadIdx.x] = A[r * N + k0 + threadIdx.x];   // coalesced
        Bs[threadIdx.y][threadIdx.x] = B[(k0 + threadIdx.y) * N + c]; // coalesced
        __syncthreads();
        for (int k = 0; k < T; ++k) acc += As[threadIdx.y][k] * Bs[k][threadIdx.x];
        __syncthreads();
    }
    C[r * N + c] = acc;     // (assumes N % T == 0 for brevity)
}
```
Each element loaded from global is reused T times ⇒ arithmetic intensity × T.

**(b) Reduction with warp shuffles:**
```cuda
__inline__ __device__ float warpSum(float v) {
    for (int off = 16; off > 0; off >>= 1) v += __shfl_down_sync(0xffffffff, v, off);
    return v;   // lane 0 holds the warp's sum
}
```
Block reduction: each warp reduces → lane 0 writes to shared memory → first warp reduces those → one atomicAdd per block.

### 2.13 GPU optimisation checklist (say this in interviews)
1. Is it **memory- or compute-bound**? (Roofline / Nsight Compute "SOL" %.)
2. Coalesce global accesses; use SoA; vectorised loads.
3. Reuse data via shared memory/registers (tiling) ⇒ raise arithmetic intensity.
4. Avoid shared-memory bank conflicts (padding / swizzling).
5. Minimise divergence; keep branches warp-uniform.
6. Enough parallelism: occupancy *or* ILP; avoid register spills.
7. Use tensor cores & lower precision where accuracy allows.
8. Fuse kernels to avoid round trips to HBM; reduce launch overhead (CUDA graphs).
9. Overlap transfers with compute (streams, pinned memory); minimise PCIe traffic.

---

## 3. Worked Numericals

**N1. Occupancy.** SM: 65,536 registers, 2048 threads, 32 blocks, 228 KB shared. Kernel: 256 threads/block, 64 registers/thread, 48 KB shared/block. (Ignore allocation granularity.)
- Registers: 65536 / (64 × 256) = **4 blocks**
- Shared memory: ⌊228 / 48⌋ = **4 blocks**
- Threads: 2048 / 256 = 8 blocks; block limit 32
- ⇒ 4 blocks = 1024 threads = 32 warps ⇒ occupancy = 32/64 = **50%**. Reducing registers to 32 doesn't help — shared memory still limits to 4.

**N2. Coalescing efficiency.** Warp reads `float a[k * s]`:
s = 1 → 4 sectors, **100%**; s = 2 → 8 sectors, **50%**; s = 8 → 32 sectors (one per thread, 4 B used of 32 B) → **12.5%**; s ≥ 8 → still 12.5% (32 sectors) but more cache lines.

**N3. Bank conflict degree.** `__shared__ float s[...]; s[threadIdx.x * k]`:
bank = (tid × k) mod 32. k = 1 → conflict-free; k = 2 → 2-way; k = 4 → 4-way; **k = 32 → 32-way**; k odd (e.g., 33) → conflict-free (gcd(k, 32) = 1). Conflict degree = gcd(k, 32).

**N4. Arithmetic intensity of tiled GEMM.** Tile T×T, FP32. Per k-step: load 2T² floats = 8T² B; compute T³ FMAs = 2T³ FLOPs ⇒ AI = 2T³ / 8T² = **T/4 FLOP/B**. Naive (T = 1): 0.25. T = 32: **8 FLOP/B**. With a ridge point of ~20 FLOP/B (FP32 CUDA cores on H100 ≈ 67 TFLOP/s / 3.35 TB/s) you still need register-level tiling as well — which is why real GEMMs tile at register, shared-memory, and L2 levels.

**N5. Latency hiding.** HBM latency 500 cycles; each warp issues 1 load then has 25 cycles of independent work before needing it; scheduler issues 1 instr/cycle. Warps needed per scheduler ≈ 500 / 25 = **20**. A sub-partition holds at most 16 warps (64/4) ⇒ latency **not fully hidden** ⇒ add ILP (e.g., each thread processes 2–4 elements, issuing loads early).

**N6. PCIe vs compute.** Copy 1 GB to the GPU over PCIe 4 (≈25 GB/s achievable) = **40 ms**. A kernel that processes it at 2 TB/s = **0.5 ms**. ⇒ the transfer dominates 80× — keep data on the GPU.

---

## 4. Interview Questions

### Basic
- **Q: CPU vs GPU design philosophy?** Latency-optimised vs throughput-optimised (Section 1).
- **Q: What is a warp? Why 32?** Group of threads issued together; 32 balances SIMD efficiency vs divergence and fits memory transaction sizes (32 × 4 B = 128 B).
- **Q: Thread, block, grid — how do they map to hardware?** (2.2.)
- **Q: What is shared memory?** On-chip, per-block, programmer-managed scratchpad.

### Intermediate
- **Q: What is warp divergence and how do you reduce it?** (2.4.)
- **Q: What is memory coalescing?** (2.6.)
- **Q: What are bank conflicts? How does padding fix the transpose?** (2.7.)
- **Q: What limits occupancy? Is 100% occupancy always best?** (2.8.)
- **Q: Why can't you call `__syncthreads()` inside an `if` that only some threads take?** Deadlock / undefined — the barrier counts all threads of the block.

### Advanced / killer follow-ups
- **Q: How does a GPU hide memory latency without OoO or big caches?** Massive multithreading with resident contexts; Little's law; also ILP and async copies (cp.async/TMA).
- **Q: Your kernel is at 30% of peak bandwidth. How do you debug?** Nsight Compute: check sectors/request (coalescing), L2 hit rate, achieved occupancy, stall reasons (long scoreboard = memory, short scoreboard = shared mem / bank conflicts), warp efficiency (divergence), register spills.
- **Q: Explain tensor cores and why BF16/FP8 help.** (2.9 + formats.) More MACs per area/energy, half/quarter bytes ⇒ also higher effective bandwidth.
- **Q: How would you implement a fast reduction?** Grid-stride loop for coalesced loads + per-thread accumulation (ILP), warp shuffles, one shared-memory stage, one atomic per block (or a second kernel).
- **Q: What is Independent Thread Scheduling and what did it break?** Per-thread PCs since Volta; old "warp-synchronous" code that assumed implicit lockstep needs `__syncwarp()` / `_sync` intrinsics.
- **Q [HFT]: Would you use a GPU for tick-to-trade?** Generally no — PCIe round-trip and kernel launch latency (µs) dominate; GPUs are for throughput (risk, backtesting, ML). Ultra-low latency uses FPGAs/ASICs or tuned CPUs.

---

## 5. Common Traps
- Saying "GPU threads are like CPU threads" — they're SIMT lanes; independence costs divergence.
- Confusing bank conflicts (shared memory) with coalescing (global memory).
- Thinking occupancy = performance.
- Ignoring PCIe transfer cost in end-to-end speedup claims.

---

## 6. Revision Summary
| | |
|---|---|
| Philosophy | CPU latency vs GPU throughput; hide latency with warps |
| Hierarchy | grid → block (one SM) → warp (32) → thread |
| SM (H100-ish) | 4 schedulers, 64K regs, 228 KB smem, 64 warps max, 4 tensor cores |
| Divergence | paths serialized with masks; make warp-uniform |
| Coalescing | 32-B sectors; consecutive threads → consecutive addresses; SoA |
| Bank conflicts | 32 banks × 4 B; degree = gcd(stride, 32); pad [32][33] |
| Occupancy | min over regs, smem, threads, blocks; not always ↑ perf |
| Latency | reg ~0, smem ~25, L2 ~200, HBM ~500 cycles |
| Tensor cores | MMA on FP16/BF16/TF32/FP8, FP32 accumulate |
| Host link | PCIe ≪ HBM ⇒ keep data on GPU, pinned memory, streams |
