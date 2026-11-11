# 06. Cache Memory ⭐ (most-asked COA topic)

> **Asked at:** **Everyone.** [NVIDIA] [APPLE] [QCOM] cache design trade-offs, address breakdown, AMAT · [HFT] [DESHAW] cache-friendly code, conflict misses, prefetching, "why is column traversal slow?" · [TI] write policies, cache vs DMA coherence · [AMZN] concept questions.
> **Prerequisites:** [02 Performance](02-Performance-Metrics-and-Laws.md).

---

## 1. Intuition
DRAM is ~100× slower than the CPU. A cache is a **small, fast memory that keeps copies of recently used data close to the core**, betting on *locality*:
- **Temporal locality:** used now ⇒ likely used again soon (loop variables, hot structs).
- **Spatial locality:** used now ⇒ neighbours likely used soon (arrays, sequential code).

Caches move data in **fixed-size blocks (lines), typically 64 B**, so one miss brings in neighbours "for free". Every question in this note is a variation of: *where can a block live, how do we find it, what do we evict, and what happens on a write?*

---

## 2. Deep Dive

### 2.1 The memory hierarchy (typical modern desktop/server numbers — vary by chip)
| Level | Size | Latency | Notes |
|---|---|---|---|
| Registers | ~1 KB (arch) / few KB (physical) | 0 cycles | |
| L1 D / L1 I | 32–64 KB each (Apple P-core: 128 KB D / 192 KB I) | **~4–5 cycles (~1 ns)** | per core, VIPT |
| L2 | 256 KB – 2 MB (Apple: 16 MB+ shared per cluster) | ~12–16 cycles | per core (usually) |
| L3 / LLC | 8 – 100+ MB | ~40–70 cycles | shared, sliced |
| DRAM | GBs | **~80–120 ns (~250–400 cycles)** | |
| NVMe SSD | TBs | ~10–100 µs | |
| HDD | TBs | ~5–10 ms | |
Rule of thumb: each level ~3–10× slower and ~4–10× larger.

### 2.2 Address breakdown — the core skill
```
 Physical/virtual address (A bits):
 ┌──────────────────────┬────────────────┬──────────────────┐
 │        TAG           │     INDEX      │   BLOCK OFFSET   │
 └──────────────────────┴────────────────┴──────────────────┘
   A − index − offset     log2(#sets)      log2(block size)

 #lines = cache size / block size
 #sets  = #lines / associativity
```
| Organisation | Ways | Sets | Index bits | Where can a block go? |
|---|---|---|---|---|
| Direct-mapped | 1 | #lines | log₂(#lines) | exactly one line |
| N-way set-associative | N | #lines/N | log₂(#lines/N) | any of N lines in one set |
| Fully associative | #lines | 1 | 0 | anywhere |

Each line stores: **valid bit, dirty bit (write-back), tag, replacement state, data**.

### 2.3 Lookup hardware (N-way)
```
 index ─► select set ─► read N tags + N data blocks in parallel
                          │
            N comparators: tag_i == addr.tag && valid_i ?
                          │
              hit? ── N:1 mux selects the hitting way ── offset selects bytes
```
Trade-offs of associativity:
| Higher associativity | |
|---|---|
| ✅ fewer **conflict** misses | |
| ❌ more comparators, bigger mux ⇒ **longer hit time**, more power | |
| ❌ replacement state grows | |
Direct-mapped is fastest (data can be used before tag check finishes — speculative use). Typical: L1 8–12-way, L2 8–16-way, L3 11–20-way.

### 2.4 The 3 Cs (+1) of misses
| Miss | Cause | Reduce by | Side effect |
|---|---|---|---|
| **Compulsory** (cold) | first touch of a block | larger blocks, **prefetching** | larger blocks ↑ miss penalty, may ↑ conflict |
| **Capacity** | working set > cache | bigger cache, blocking/tiling | bigger ⇒ slower hit, more power |
| **Conflict** | too many blocks map to the same set | higher associativity, victim cache, padding | slower hit |
| **Coherence** (4th C) | invalidated by another core's write | avoid sharing / false sharing | see [10](10-Cache-Coherence-and-Memory-Consistency.md) |
Classification trick: a miss that would **hit in a fully-associative cache of the same size** is a conflict miss; if it misses there too (and not the first access), it's capacity.

**Block size curve:** miss rate first falls (spatial locality), then rises (fewer blocks ⇒ more conflict/capacity misses, pollution). Miss penalty grows with block size.

### 2.5 Replacement policies
| Policy | Notes |
|---|---|
| LRU | ideal-ish for recency; true LRU for N ways needs log₂(N!) bits/set — costly beyond 4–8 ways |
| **Pseudo-LRU (tree-PLRU)** | N−1 bits per set; each bit points to the less-recently-used half; used in real L1s |
| FIFO | simple, Belady's anomaly possible |
| Random | surprisingly good for large associativity, no state |
| RRIP / SRRIP / DRRIP | predicts re-reference interval; **scan-resistant** — used in modern LLCs |
| Belady's OPT | evict the block used farthest in future — theoretical bound |
LRU's weakness: a streaming scan larger than the cache evicts everything useful (thrashing on cyclic access of size cache+1 ⇒ 0% hit rate).

### 2.6 Write policies
| On write **hit** | |
|---|---|
| **Write-through** | update cache *and* next level. Simple, next level always up to date, high traffic → needs a **write buffer** |
| **Write-back** | update cache only, set **dirty**; write the block to next level on eviction. Less traffic; standard for L1/L2/L3 in CPUs |

| On write **miss** | |
|---|---|
| **Write-allocate** (fetch-on-write) | bring the block in, then write — pairs with write-back |
| **No-write-allocate** (write-around) | write straight to next level — pairs with write-through |

- **Write buffer:** lets the CPU continue after a store; read misses must check the buffer (or wait) for RAW correctness. *Merging* write buffers combine writes to the same block.
- **Streaming / non-temporal stores** (`movntdq`, `_mm_stream_si128`) bypass the cache to avoid polluting it with data you won't reread.

### 2.6.1 Putting it together: what happens on a read and on a write, step by step
This section combines §2.2 (address split), §2.3 (tag check), §2.5 (replacement) and §2.6 (write policies) into one flow. We use **write-back + write-allocate**, which is what CPU caches use. The write-through variant is at the end.

**Two facts to keep in mind:**
1. The CPU reads or writes **1–8 bytes** at a time (a load or store), but the cache moves data to and from the next level only in **whole blocks** (e.g., 64 B).
2. All of this is done by the **cache controller in hardware**. The program just executes `lw` / `sw`; it never sees hits or misses (except as speed).

**What one cache line holds:**
```
 ┌───┬───┬───────┬──────────────────────────────────┐
 │ V │ D │  Tag  │  Data: the whole block (e.g. 64 B) │
 └───┴───┴───────┴──────────────────────────────────┘
 V = valid (line holds real data)   D = dirty (line was modified; memory is out of date)
```

#### A. READ (load), step by step
```
          CPU: load from address A
                    │
   1. Split A into TAG | INDEX | OFFSET
                    │
   2. INDEX selects the set (the one line, in a direct-mapped cache)
                    │
   3. Compare: valid == 1  AND  stored tag == TAG ?
            ┌───────┴────────┐
          YES                NO
          (HIT)             (MISS)
            │                 │
   4. OFFSET picks the     5a. Choose a victim line in the set
      bytes inside the         (direct-mapped: the only line; N-way: replacement policy, e.g. LRU)
      block → send to CPU 5b. Victim valid AND dirty?  → write the victim's WHOLE block
      (~4 cycles at L1)        back to the next level first (otherwise just overwrite it)
                          5c. Request the WHOLE block containing A from the next level
                              (L2 → L3 → DRAM: each level repeats this same procedure)
                          5d. Install it: data ← block, tag ← TAG, V = 1, D = 0
                          5e. OFFSET picks the bytes → send to CPU
                              (often the requested word is sent first: "critical word first")
```

#### B. WRITE (store), step by step (write-back + write-allocate)
```
          CPU: store value X to address A
                    │
   Steps 1–3 exactly as for a read (split address, select set, compare tag)
            ┌───────┴────────┐
          YES                NO
          (HIT)             (MISS)
            │                 │
   4. Write X into the     5. WRITE-ALLOCATE: do steps 5a–5d of a read miss
      line at OFFSET          (evict victim, write it back if dirty, fetch the whole block, install it)
      Set D = 1            6. Now it's a hit: write X into the line at OFFSET, set D = 1
      Memory is NOT
      updated now (it gets the new data only when this line is evicted)
```
**Why fetch the whole block on a write miss when you only write 4 bytes?** The line must hold a complete, correct block. The other 60 bytes have to come from memory, because the cache keeps one valid bit per line, not one per byte.

#### C. Worked trace with a tiny cache
**Cache:** direct-mapped, **4 lines**, **16-byte blocks**, write-back + write-allocate. **8-bit addresses.**
- Offset = log₂16 = **4 bits** (bits 3–0), index = log₂4 = **2 bits** (bits 5–4), tag = 8 − 4 − 2 = **2 bits** (bits 7–6).

| Address | Binary (tag \| index \| offset) | Tag | Line (index) | Offset | Block covers |
|---|---|---|---|---|---|
| 0x34 | 00 \| 11 \| 0100 | 00 | 3 | 4 | 0x30–0x3F |
| 0x38 | 00 \| 11 \| 1000 | 00 | 3 | 8 | 0x30–0x3F (same block) |
| 0x3C | 00 \| 11 \| 1100 | 00 | 3 | 12 | 0x30–0x3F (same block) |
| 0x74 | 01 \| 11 \| 0100 | 01 | 3 | 4 | 0x70–0x7F (different block, **same line**) |

**Operations:** read 0x34 → read 0x38 → write 0x3C → read 0x74 → read 0x34. All of them use line 3. Start: all lines invalid.

| # | Operation | Line 3 before (V, D, Tag) | Result | What the cache does | Line 3 after (V, D, Tag) | Traffic to memory |
|---|---|---|---|---|---|---|
| 1 | Read 0x34 | 0, 0, — | **Miss** (compulsory: line invalid) | victim invalid → nothing to write back; fetch block 0x30–0x3F; install; return byte at offset 4 | 1, 0, 00 | fetch 16 B |
| 2 | Read 0x38 | 1, 0, 00 | **Hit** (tag 00 matches) | return byte at offset 8. Spatial locality: step 1 already brought this byte in | 1, 0, 00 | none |
| 3 | Write 0x3C | 1, 0, 00 | **Hit** | write new value at offset 12 **in the cache only**; set D = 1. Memory now holds the old value (stale), which is fine: the cache has the latest copy | 1, **1**, 00 | none |
| 4 | Read 0x74 | 1, 1, 00 | **Miss** (conflict: tag 01 ≠ 00) | victim is **dirty** → first **write back** block 0x30–0x3F (memory now gets step 3's value); then fetch block 0x70–0x7F; install; return offset 4 | 1, 0, 01 | write 16 B + fetch 16 B |
| 5 | Read 0x34 | 1, 0, 01 | **Miss** (conflict: tag 00 ≠ 01) | victim is **clean** → just overwrite, no write-back; fetch block 0x30–0x3F (it includes step 3's value, thanks to the write-back in step 4); install | 1, 0, 00 | fetch 16 B |

**Totals:** 2 hits, 3 misses, 3 block fetches, 1 write-back.
**What-ifs:**
- **2-way set-associative** (same 64 B total → 2 sets): blocks 0x30 and 0x70 could both live in the same set, so step 5 would **hit** and step 4 wouldn't evict anything. Associativity reduces conflict misses (§2.4).
- **Write-through** (described next): step 3 would also write to memory immediately, the line would never be dirty, and step 4 would need no write-back.

#### D. The write-through variant (with no-write-allocate)
| Case | Write-back + write-allocate (above) | Write-through + no-write-allocate |
|---|---|---|
| Write **hit** | update cache only, set dirty | update cache **and** send the write to the next level (through a write buffer, so the CPU doesn't wait) |
| Write **miss** | fetch block into cache, then write it | send the write **only** to the next level; the cache is not changed |
| Eviction | dirty line must be written back | never needs a write-back (memory is always up to date) |
| Dirty bit | needed | not needed |
Reads behave the same in both, except that write-through victims never need a write-back.

#### E. With several cache levels
An L1 miss sends the request to **L2**, which does **its own** lookup (with its own tag/index/offset split, since its size and associativity differ). On an L2 miss it goes to L3, then DRAM. A dirty block evicted from L1 is written back into **L2**, not directly to DRAM. Each level only talks to the level right below it. **Full step-by-step explanation in §2.6.2.**

#### F. Common confusions
- **"Write-back" vs "write-allocate" are answers to different questions.** Write-back/through: *on a write hit, when does the next level get updated?* (later vs now). Write-allocate/no-allocate: *on a write miss, do we bring the block into the cache?* (yes vs no).
- **Stale memory is normal with write-back.** For one core that's fine: the cache has the newest data. For other cores, coherence handles it ([10](10-Cache-Coherence-and-Memory-Consistency.md)); for DMA devices, the software must clean/invalidate ([13](13-IO-Interrupts-DMA.md)).
- **Only the tag is stored and compared**, not the full address. The index is implied by which line you're looking at, and the offset only selects bytes within the block.
- **A miss always moves a whole block**, even for a 1-byte load. That's why sequential access is fast (one miss, then up to 63 hits) and random access is slow.
- **The CPU waits on a miss** (in-order pipeline: stall), or keeps executing independent work (out-of-order core with a non-blocking cache, §2.9).

**Quick self-check**
1. In the trace, why was step 2 a hit even though 0x38 had never been accessed before?
2. Why did step 4 need two memory transfers but step 5 only one?
3. With write-through + no-write-allocate, would step 3 change line 3 at all? (Assume the line is valid with tag 00, as in the trace.)

<details><summary>Answers</summary>

1. Step 1 fetched the whole 16-byte block 0x30–0x3F, which includes 0x38 (spatial locality).
2. In step 4 the victim was dirty (modified in step 3), so it had to be written back before the new block was fetched. In step 5 the victim was clean, so it was simply overwritten.
3. Yes. It's a write **hit**, so the new value is written into the cache line **and** sent to memory. (No-write-allocate only matters on a write **miss**.) The line stays clean (D is not used).
</details>

### 2.6.2 Reads and writes across multiple levels (L1 → L2 → L3 → DRAM)
**Typical hierarchy assumed below** (desktop/server CPU; numbers are typical, not exact):
| Level | Example size / organisation | Private or shared | Hit latency | Write policy |
|---|---|---|---|---|
| L1-D | 32 KB, 8-way, 64 B lines | private (per core) | ~4 cycles | write-back + write-allocate |
| L2 | 1 MB, 16-way, 64 B lines | private (per core) | ~12–14 cycles | write-back + write-allocate |
| L3 | 8 MB, 16-way, 64 B lines | shared by all cores | ~40–50 cycles | write-back |
| DRAM | GBs | shared | ~200–300 cycles (~80–100 ns) | — |

#### A. Rule 1: each level is a separate cache with its own lookup
All levels use the same **block size (64 B)**, so they move the same blocks. But each level has a different number of sets, so the **same address splits differently** at each level.

Example: physical address **0x1234ABCD** (32-bit):
| Level | Sets | Offset bits | Index bits | Tag bits | Set used for 0x1234ABCD |
|---|---|---|---|---|---|
| L1 (32 KB, 8-way) | 32 KB / 64 B / 8 = **64** | 6 (bits 5–0) | 6 (bits 11–6) | 20 (bits 31–12) | bits 11–6 = 101111₂ = **47** |
| L2 (1 MB, 16-way) | 1 MB / 64 B / 16 = **1024** | 6 | 10 (bits 15–6) | 16 (bits 31–16) | bits 15–6 = 1010101111₂ = **687** |
| L3 (8 MB, 16-way) | 8 MB / 64 B / 16 = **8192** | 6 | 13 (bits 18–6) | 13 (bits 31–19) | bits 18–6 (its own set) |
The offset (0x0D = byte 13 of the block) is the same everywhere, because the block size is the same. The block itself is **0x1234ABC0 – 0x1234ABFF** at every level.

#### B. Rule 2: a READ goes down level by level until it finds the block; the block comes back up and is installed on the way
```
 CPU: load from 0x1234ABCD
   │
   ▼  ① L1 lookup (set 47) ───── hit? → return bytes (~4 cycles). DONE
   │     miss: reserve a miss buffer (MSHR); request block 0x1234ABC0 from L2
   ▼  ② L2 lookup (set 687) ──── hit? → send the 64 B block up to L1 (~14 cycles total). DONE
   │     miss: request the block from L3
   ▼  ③ L3 lookup ───────────── hit? → send block up to L2 and L1 (~40–50 cycles total). DONE
   │     miss: request the block from the memory controller
   ▼  ④ DRAM access ─────────── block read (~200–300 cycles total)
   │
   ▲  ⑤ The block travels back UP: installed in L3 → L2 → L1 (depending on inclusion policy, part F),
         and the requested bytes are given to the CPU (critical word first)
```
**At every level where the block gets installed, the read-miss steps from §2.6.1 happen locally:** choose a victim in that level's set; if the victim is dirty, write it back to the **next level down**; then install the new block (V = 1, D = 0).

| Where the block is found | What happens | Typical total latency |
|---|---|---|
| L1 hit | L1 returns the bytes | ~4 cycles |
| L1 miss, **L2 hit** | L2 sends the block to L1; L1 evicts a victim (dirty → written to L2), installs, returns bytes | ~12–14 cycles |
| L1, L2 miss, **L3 hit** | L3 → L2 → L1; L2 and L1 each evict a victim | ~40–50 cycles |
| Miss in all caches | DRAM → L3 → L2 → L1; up to three victims evicted | ~200–300 cycles |
Latencies add up because each level must discover its own miss before asking the next level. This is exactly what the multilevel AMAT formula computes (§2.7):
```
AMAT = HT₁ + MR₁ × ( HT₂ + MR₂ × ( HT₃ + MR₃ × T_DRAM ) )      (MR₂, MR₃ = local miss rates)
```

#### C. Rule 3: a WRITE only touches L1 (if it hits); a write miss first READS the block in
With write-back + write-allocate at every level:
| Case | What happens |
|---|---|
| **Write hit in L1** | Write the bytes into L1, set **dirty** in L1. **L2, L3 and DRAM are not touched.** Their copies (if any) are now old |
| **Write miss in L1** | Write-allocate: fetch the block exactly like a read miss (rule 2: from L2, L3 or DRAM, wherever it's found), install it in L1, **then** write the bytes and set dirty in L1 |
So a store that misses is "a read miss followed by a write hit". The new value lives **only in L1**.

Two practical details:
- **The CPU doesn't wait for a store.** The store's address and value go into the core's **store buffer**, and the core continues. The store is written into L1 later, once the **cache line (64 B block) containing its address is present in L1 and writable**. Example: `sw x5, 0(x6)` with `x6 = 0x1000` needs block 0x1000–0x103F:
  ```
  1. Store executes → (address 0x1000, value) placed in store buffer → core moves on
  2. Store commits and is the oldest store in the buffer
  3. Is block 0x1000–0x103F in L1 and writable?
       YES → write bytes into L1, mark line dirty, free buffer entry (a few cycles)
       NO  → L1 fetches the block from L2/L3/DRAM (write-allocate); the store WAITS in the
             buffer (~14–300 cycles) while the core keeps executing; block arrives → installed
             → store writes into it → entry freed
  ```
  | A store leaves the buffer only when… | Why |
  |---|---|
  | it has **committed** | speculative stores must never reach the cache ([09](09-ILP-Superscalar-Out-of-Order.md)) |
  | it's the **oldest** store in the buffer | x86 writes stores to L1 in program order (TSO, [10](10-Cache-Coherence-and-Memory-Consistency.md)) |
  | its line is **in L1 and writable** | on a multicore, "writable" = M or E state; other cores' copies must be invalidated first ([10](10-Cache-Coherence-and-Memory-Consistency.md)) |

  **Condition 1 explained (committed).** CPUs run instructions *before* knowing they should run: at an `if`, they guess the direction and keep going (speculation).
  ```c
  if (i < size)   // CPU guesses "true", but the real answer is false
      a[i] = 5;   // store runs early, based on the guess
  ```
  If this store reached the cache and the guess was wrong, memory would hold a write the program never made, and it couldn't be undone (another core may already have read it). Registers are private and can be rolled back; the shared cache can't. So the store waits in the private store buffer. **Commit** = the CPU has confirmed the instruction is really part of the execution (earlier branches guessed right, no earlier instruction faulted). Right guess → commit → write to L1. Wrong guess → the buffer entry is deleted, no harm. *Analogy:* the store buffer is an email draft; writing the cache is pressing Send.

  **Condition 2 explained (oldest first, x86).** Several committed stores may sit in the buffer, ready at different times. Writing the ready one first would let other cores see them out of order:
  ```c
  // Core 0                              // Core 1
  data  = 42;   // line misses in L1     while (ready == 0) { }
  ready = 1;    // line hits in L1       print(data);   // expects 42
  ```
  If `ready = 1` reached the cache first, Core 1 could see `ready == 1` while `data` is still 0, and print 0. x86 promises other cores see a core's stores in program order (TSO), so the buffer drains oldest first: `ready = 1` waits for `data = 42`. Your own thread never notices the order (its loads check the store buffer first); only other cores and devices do. **ARM and RISC-V don't promise this**: stores may reach the cache out of order, so use a barrier or `std::atomic` release/acquire when order matters (portable C++: always use `std::atomic` for such flags).
  Meanwhile, a later load from the same address gets the value straight from the store buffer (**store-to-load forwarding**), so the thread sees its own store immediately. If the buffer **fills** (it has tens of entries), the core stalls; this can happen during long runs of store misses (e.g., `memset` on a huge buffer). Loads are what the CPU normally waits for.
- **On a multicore**, the write miss asks for the block **with ownership** (a *read-for-ownership*), which also invalidates other cores' copies. That's coherence ([10](10-Cache-Coherence-and-Memory-Consistency.md)).

#### D. Rule 4: dirty data moves down only when it is evicted, one level at a time
```
 L1 evicts a dirty line  ──► written into L2   (L2's copy updated, marked dirty in L2)
 L2 evicts a dirty line  ──► written into L3   (marked dirty in L3)
 L3 evicts a dirty line  ──► written to DRAM   (DRAM finally up to date)
 Clean lines are simply dropped at any level: a correct copy exists below.
```

#### E. Worked trace: where does the newest value live?
A variable `count` lives in block B. Initially `count = 0`, and B is only in DRAM. Assume an inclusive hierarchy (a block in L1 is also in L2 and L3).

| Time | Event | L1 | L2 | L3 | DRAM | Memory-system traffic |
|---|---|---|---|---|---|---|
| t1 | read `count` | 0 (clean) | 0 (clean) | 0 (clean) | 0 | miss everywhere: DRAM → L3 → L2 → L1 |
| t2 | `count = 1` | **1 (dirty)** | 0 | 0 | 0 | none: write hit in L1 |
| t3 | `count = 2` | **2 (dirty)** | 0 | 0 | 0 | none |
| t4 | L1 needs the space, evicts B | — | **2 (dirty)** | 0 | 0 | L1 → L2 write-back |
| t5 | read `count` | 2 (clean) | 2 (dirty) | 0 | 0 | L1 miss, **L2 hit** → returns 2 ✅ |
| t6 | L1 evicts B again | — | 2 (dirty) | 0 | 0 | none: L1's copy was clean, just dropped |
| t7 | L2 evicts B | — | — | **2 (dirty)** | 0 | L2 → L3 write-back |
| t8 | L3 evicts B | — | — | — | **2** | L3 → DRAM write-back |

**Key insight: the newest copy is always the one closest to the CPU.** Lower levels may hold old values, but a lookup always searches **top-down** (L1 → L2 → L3 → DRAM) and stops at the first hit, so it always finds the newest copy first (t5 got 2, not the stale 0 in L3 and DRAM). Two writes (t2, t3) cost zero traffic. DRAM was written only once, at t8. That's why write-back is used at every level.
(Other cores and DMA devices don't search *your* L1 and L2 that way, which is why coherence ([10](10-Cache-Coherence-and-Memory-Consistency.md)) and DMA cache maintenance ([13](13-IO-Interrupts-DMA.md)) exist.)

#### F. How the inclusion policy changes fills and evictions (see §2.8)
| Policy | On a fill from DRAM | When L3 evicts a block | On an L3 hit |
|---|---|---|---|
| **Inclusive** | install in L3, L2 and L1 | must also **invalidate** that block in L1/L2 (back-invalidation), writing back dirty data first | copy to L2/L1; L3 keeps it |
| **Exclusive** | install in L1/L2 only, not in L3 | — (L3 holds only blocks evicted from L2) | **move** the block up to L2/L1 and remove it from L3 |
| **NINE** (non-inclusive, non-exclusive) | usually install in L2/L1, L3 optional | no back-invalidation needed | copy up; L3 may keep it |
In an exclusive design, L3 works like a big **victim cache**: blocks evicted from L2 (clean or dirty) are placed into L3.

#### G. Variant: write-through L1 + write-back L2
Some designs (e.g., many GPUs, some embedded and older CPUs) make L1 **write-through** to L2 and keep L2/L3 write-back:
- every store also goes to L2 (L1 is never dirty, so L1 evictions are free and L1 coherence is simple);
- L2 absorbs the store traffic, so DRAM still sees writes only on L2/L3 evictions.

**Quick self-check**
1. A load misses in L1 and L2 but hits in L3. In which levels can a victim be evicted, and where does a dirty L1 victim go?
2. In the trace, at t3 which levels hold an out-of-date copy of `count`? Why is the program still correct?
3. A store misses in L1 and hits in L2. Does the new value get written into L2?

<details><summary>Answers</summary>

1. L2 and L1 (L3 keeps its copy on a hit). A dirty L1 victim is written into **L2**, not DRAM.
2. L2, L3 and DRAM all hold 0 while L1 holds 2. It's correct because every access searches L1 first and finds the newest value there.
3. No. L2 sends the block up to L1 (write-allocate), and the store is done in L1, which becomes dirty. L2 keeps the old value until L1 later evicts the dirty line back into it.
</details>

### 2.7 AMAT and multilevel caches
```
AMAT = Hit time + Miss rate × Miss penalty

Two levels:
AMAT = HT_L1 + MR_L1 × (HT_L2 + MR_L2(local) × MP_L2)

Local miss rate  (L2) = L2 misses / L2 accesses   (L2 sees only L1 misses → looks bad)
Global miss rate (L2) = L2 misses / CPU accesses  = MR_L1 × MR_L2(local)

Memory stall cycles per instruction
  = (memory accesses / instr) × miss rate × miss penalty
CPI = CPI_base + stall cycles per instruction
```
**Out-of-order cores overlap misses** (memory-level parallelism), so real stall cost < AMAT model — mention this as the "senior" caveat.

### 2.8 Inclusion policies
| Policy | Meaning | Pro | Con |
|---|---|---|---|
| Inclusive | everything in L1/L2 is also in L3 | L3 acts as a **snoop filter** for coherence | wasted capacity; L3 eviction forces back-invalidation of L1/L2 |
| Exclusive | a block lives in exactly one level | max effective capacity (AMD's L3 has historically been mostly a victim cache) | more data movement |
| NINE (non-inclusive non-exclusive) | no guarantee | flexible | needs separate snoop filter (Intel Skylake-SP onward) |

### 2.9 Cache optimisations (H&P list — know names + one line each)
**Reduce hit time:** small & simple L1, **way prediction**, VIPT (avoid translation, [07](07-Virtual-Memory-and-TLB.md)), pipelined cache access.
**Increase bandwidth:** **non-blocking caches** (hit-under-miss, miss-under-miss using MSHRs), multibanked caches, pipelining.
**Reduce miss penalty:** **critical word first / early restart**, multilevel caches, give **reads priority** over writes in the write buffer, merging write buffer, **victim cache** (small FA buffer of recently evicted lines → kills conflict misses).
**Reduce miss rate:** larger blocks, larger caches, higher associativity, **compiler optimisations** (loop interchange, blocking, fusion, data layout).
**Hide latency via parallelism:** **hardware prefetching** (next-line, stride, stream prefetchers), **software prefetching** (`__builtin_prefetch`).

### 2.10 Writing cache-friendly code [HFT][NVIDIA][DESHAW]
**(a) Traversal order — C is row-major.**
```c
int a[N][N];
for (i) for (j) sum += a[i][j];   // stride-1: one miss per 16 ints (64B line) ✅
for (j) for (i) sum += a[i][j];   // stride-N*4 bytes: potentially a miss per access ❌
```
**(b) Loop blocking / tiling (matrix multiply).**
```c
for (ii = 0; ii < N; ii += B)
  for (jj = 0; jj < N; jj += B)
    for (kk = 0; kk < N; kk += B)
      for (i = ii; i < ii+B; i++)
        for (k = kk; k < kk+B; k++) {
          double r = A[i][k];
          for (j = jj; j < jj+B; j++)
            C[i][j] += r * Bm[k][j];
        }
// choose B so three B×B tiles fit in cache: 3·B²·8 bytes ≤ cache size
```
Naive matmul touches O(N³) cache lines; tiled ≈ O(N³ / B) — the same trick GPUs use with shared memory.

**(c) Data layout: AoS vs SoA.**
```c
struct Order { double px; int qty; char side; char pad[51]; };  // AoS: scanning px touches 64B/order
struct Orders { double px[N]; int qty[N]; char side[N]; };       // SoA: scanning px is dense & SIMD-friendly
```
Hot/cold splitting: keep frequently used fields together in one cache line, move cold fields elsewhere.

**(d) Pointer chasing** (linked lists, trees with heap nodes) defeats prefetchers and serialises misses → prefer contiguous arrays (`std::vector`, flat hash maps, B-trees with wide nodes).

**(e) Power-of-two stride conflicts (4K aliasing).** L1 32 KB, 8-way, 64 B ⇒ 64 sets ⇒ addresses 4096 B apart map to the **same set**; touching 9+ such addresses thrashes an 8-way set. Pad arrays/rows (e.g., row length 1024+16) to break the pattern.

**(f) Lines are the unit of everything:** align hot structures to 64 B, avoid **false sharing** between threads ([10](10-Cache-Coherence-and-Memory-Consistency.md)).

### 2.11 How the cache levels are connected (is there a "bus"?)
Every level is connected by wires. Whether a connection counts as a **bus** depends on whether it is a **shared** set of wires that several units take turns on (with arbitration), or a **dedicated point-to-point link**.

| Connection | Typical modern implementation | Shared (bus-like)? |
|---|---|---|
| Core ↔ L1-I / L1-D | dedicated ports (fetch unit ↔ L1-I; load/store unit ↔ L1-D) | no |
| L1-I / L1-D ↔ L2 (private per core) | **dedicated links** from each L1 into L2's request port(s), plus an **arbiter** when both miss together; wide data path (e.g., ~64 B/cycle) | a little: two requesters share L2's port |
| L2 ↔ L3 / other cores | **on-chip interconnect**: ring (Intel client), mesh (Intel server), Infinity Fabric (AMD between chiplets), cluster fabric (Apple: L2 shared by a core cluster) | yes: a shared network with packets and arbitration |
| L3 / memory controller ↔ DRAM | **DDR memory channel**: a real external bus | yes |
| CPU ↔ devices | PCIe: point-to-point serial links | no (switched) |
History: in the 1990s, L2 was a separate chip off the CPU die. The Pentium Pro/II reached it over a dedicated **"backside bus"**, and memory over the **"front-side bus"**. Moving L2 onto the die (Pentium III "Coppermine", ~1999) made it much faster.

### 2.12 Split L1, unified L2: why not split L2 too?
Almost every CPU uses **split L1 (I + D)** and **unified L2/L3**. The reason is that the trade-off flips between levels:

| | At **L1** | At **L2** |
|---|---|---|
| How often accessed | **every cycle** by both IF and MEM | only on **L1 misses** (a few % of cycles) |
| Port conflict if unified | constant structural hazard ⇒ would need an expensive 2-ported cache | rare (both L1s must miss in the same cycle) ⇒ an arbiter + 1-cycle wait is enough |
| Placement | I-cache sits next to fetch, D-cache next to load/store unit ⇒ short wires, fast hits | one block; a few extra cycles of wire don't matter for a ~12-cycle access |
| Specialisation | I-cache is read-only (no dirty bits, no store logic), tuned for sequential fetch | no benefit from specialising |
| **Verdict** | **split wins** | **unified wins** |

**Why unified wins at L2 (the interview answer):**
1. **Flexible capacity, lower miss rate.** A unified cache gives space to whichever side needs it at the moment. With a fixed split, one half can sit half-empty while the other thrashes.
   | Workload | Code footprint | Data footprint | Split L2 (1 MB I + 1 MB D) | Unified L2 (2 MB) |
   |---|---|---|---|---|
   | Web server / database | 1.5 MB | 0.5 MB | code doesn't fit → I-misses; 0.5 MB of D-space wasted | everything fits |
   | Matrix kernel | 50 KB | 1.9 MB | data doesn't fit → D-misses; ~1 MB of I-space wasted | everything fits |
   Same total SRAM, but the unified cache serves both workloads.
2. **Conflicts are rare and cheap at L2.** Only simultaneous L1-I and L1-D misses collide. An arbiter delays one by ~1 cycle on a ~12-cycle access, and a pipelined L2 overlaps them ([04 §2.7](04-Pipelining-and-Hazards.md)).
3. **Code is sometimes data.** JIT compilers (Java, JavaScript V8), program loaders, dynamic linkers and self-modifying code **write** instructions through the D-side, then **fetch** them through the I-side. ARM literal pools also put constants inside code regions. With a unified L2, the I-side finds the freshly written line right there. With separate L2s, you'd need coherence between them, or a round trip to L3/memory.
4. **Less hardware.** One set of tags, one controller, one port into L3 and the interconnect, instead of two of each.

**The exception that proves the rule:** Intel **Itanium 2 "Montecito"** had a **split L2** (1 MB L2-I + 256 KB L2-D per core), because Itanium's VLIW code was unusually large. It's an architecture-specific choice, not the norm.

**One-line answer:** *"Split at L1 because both IF and MEM hit it every cycle and need parallel, specialised, nearby ports; unify at L2 because it's accessed only on misses, so port conflicts are rare, and a shared pool uses capacity better and handles code-as-data naturally."*

---

## 3. Worked Numericals

**N1. Address breakdown.** 32-bit address, 64 KB cache, 64 B blocks, 4-way.
- Lines = 64 KB / 64 B = 1024; sets = 1024/4 = 256
- Offset = 6, Index = 8, **Tag = 32 − 8 − 6 = 18 bits**.
- Address 0x1234ABCD: offset = 0x0D (low 6 bits = 001101), index = bits[13:6] = (0xABCD >> 6) & 0xFF = 0x2AF & 0xFF = **0xAF**, tag = 0x1234ABCD >> 14 = **0x48D2**.

**N2. Tag storage overhead** for N1 (write-back, ignore replacement bits): per line 18 + 1 (V) + 1 (D) = 20 bits ⇒ 1024 × 20 = 20,480 bits = **2.5 KB** (≈ 3.9% over 64 KB data).

**N3. Hit/miss trace (P&H classic).** 4 one-word blocks; block address sequence **0, 8, 0, 6, 8**.

| Access | Direct-mapped (idx = addr mod 4) | 2-way SA, LRU (set = addr mod 2) | Fully assoc. LRU |
|---|---|---|---|
| 0 | miss (idx0←0) | miss (set0: 0) | miss |
| 8 | miss (idx0←8, evict 0) | miss (set0: 0,8) | miss |
| 0 | miss (evict 8) | **hit** | **hit** |
| 6 | miss (idx2←6) | miss (set0 full, evict LRU 8) | miss |
| 8 | miss (evict 0) | miss (evict LRU 0) | **hit** |
| **Misses** | **5** | **4** | **3** |
In DM, misses 3 and 5 are conflict misses (they hit in FA).

**N4. Two-level AMAT.** L1: hit 1 cycle, miss rate 5%. L2: hit 10 cycles, local miss rate 20%. Memory 100 cycles.
AMAT = 1 + 0.05 × (10 + 0.2 × 100) = 1 + 0.05 × 30 = **2.5 cycles**. Global L2 miss rate = 0.05 × 0.2 = **1%**.
Without L2: 1 + 0.05 × 100 = 6 cycles ⇒ L2 gives **2.4×** better AMAT.

**N5. CPI with cache stalls.** Base CPI 1. Per instruction: 1 instruction fetch + 0.36 data accesses. I-miss 2%, D-miss 5%, penalty 100 cycles.
Stalls = 1 × 0.02 × 100 + 0.36 × 0.05 × 100 = 2 + 1.8 = 3.8 ⇒ **CPI = 4.8** (a perfect cache would be 4.8× faster).

**N6. Traversal miss rates.** `int a[1024][1024]` (4 MB), L1 = 32 KB, 64 B lines (16 ints).
- Row-wise: 1 miss per 16 accesses ⇒ **6.25%**.
- Column-wise: consecutive accesses are 4 KB apart ⇒ each touches a new line; a column needs 1024 lines (64 KB) > 32 KB, and they all fall into the same few sets (4 KB stride!) ⇒ by the time you return for the next column, lines are evicted ⇒ **~100% miss**.

**N7. Write-through vs write-back traffic.** A loop writes the same 8-byte variable 1,000,000 times (always hits).
- Write-through: 1,000,000 writes of 8 B to L2 = **8 MB** traffic.
- Write-back: block written back once on eviction = **64 B**.

**N8. Tiling block size.** L1 = 32 KB, doubles (8 B). Need 3 tiles: 3 × B² × 8 ≤ 32768 ⇒ B² ≤ 1365 ⇒ **B ≈ 32** (B = 36 max; pick 32 for alignment).

**N9. Associativity from constraints (GATE style).** 16 KB 4-way cache, 32-bit address, tag = 20 bits. Block size?
Index + offset = 12 bits. Sets × block = 16 KB / 4 = 4 KB = 2¹² ✓ (consistent for any block size). Need more info — interviewers love this: **block size cannot be determined from these alone**; index + offset = log₂(cache size / ways) always.

---

## 4. Interview Questions

### Basic
- **Q: Why do caches work?** Temporal + spatial locality.
- **Q: Direct-mapped vs set-associative vs fully associative?** (Table 2.2 + trade-off 2.3.)
- **Q: Write-through vs write-back?** (2.6.) Which pairs with write-allocate? Write-back.
- **Q: What is AMAT?** HT + MR × MP, recursive for multiple levels.

### Intermediate
- **Q: Explain the 3 Cs and how each design parameter affects them.** (2.4.)
- **Q: Why is L1 small and L2 large?** L1 must hit in ~4 cycles to keep pipeline fed (latency-critical); L2 reduces miss penalty (capacity-critical). Larger arrays = longer wires = slower.
- **Q: Why are L1 caches split (I/D) but L2 unified? Why not split L2 too?** L1 is hit by IF and MEM every cycle, so splitting gives two parallel, specialised, nearby ports. L2 sees only misses, so conflicts are rare; a unified pool adapts capacity to code vs data, handles code-as-data (JIT, loaders), and needs less hardware. See §2.12.
- **Q: Why does increasing block size eventually increase miss rate?** Fewer blocks in the same capacity ⇒ more conflict/capacity misses, and unused bytes pollute the cache.
- **Q: What is a victim cache?** Small fully-associative buffer holding recently evicted lines; checked on miss; captures conflict misses of DM/low-assoc L1 (Jouppi).
- **Q [TI]: What's the problem with caches and DMA?** DMA writes memory behind the cache ⇒ stale cached data; DMA reads memory before dirty lines are written back ⇒ stale data out. Fix: cache clean/invalidate around DMA, or non-cacheable buffers, or coherent DMA (see [13](13-IO-Interrupts-DMA.md)).
- **Q [HFT]: Why is iterating a `std::list` slower than a `std::vector` even with the same Big-O?** Pointer chasing ⇒ each node a likely cache miss, no spatial locality, prefetchers can't predict, misses serialised.

### Advanced / killer follow-ups
- **Q [NVIDIA][APPLE]: How does a non-blocking cache work?** MSHRs (miss status holding registers) track outstanding misses; subsequent hits are served (hit-under-miss) and further misses allocate new MSHRs (miss-under-miss); secondary misses to the same line merge into the existing MSHR. #MSHRs limits memory-level parallelism (Little's law).
- **Q: Your code got 3× slower when you changed an array dimension from 1000 to 1024. Why?** Power-of-two stride ⇒ all rows map to the same sets ⇒ conflict misses (also 4K aliasing in load/store disambiguation on x86). Pad the row.
- **Q: Design a cache replacement policy that's scan-resistant.** RRIP: insert new lines with "distant" re-reference prediction; promote on hit; so a one-time scan is evicted first.
- **Q: Inclusive vs exclusive LLC — trade-offs?** (2.8.) Inclusive simplifies coherence (snoop filter) but back-invalidations hurt; exclusive maximises capacity.
- **Q: How does the hardware prefetcher detect strides?** Per-PC table (reference prediction table) records last address and stride; after confidence builds, issues prefetches d strides ahead. Stream prefetchers detect sequential lines in a page; prefetchers don't cross 4 KB page boundaries (physical addresses).
- **Q [HFT]: When can prefetching hurt?** Pollution (evicts useful lines), wasted bandwidth when bandwidth-bound, prefetching too late/early.
- **Q: How many cycles does an L1 hit take and why can't it be 1?** ~4–5: address generation, TLB lookup in parallel with set indexing, tag compare, way mux, data alignment/sign-extension, wire delay at GHz frequencies.

---

## 5. Common Traps
- Computing index bits from #lines instead of #sets.
- Forgetting the valid bit / dirty bit in overhead questions.
- Mixing local and global L2 miss rate in AMAT.
- Saying "bigger block always better".
- Assuming column traversal is "a bit slower" — it can be 10–50× slower.
- Forgetting instruction fetches are memory accesses too (N5).

---

## 6. Revision Summary
| | |
|---|---|
| Address | tag \| index = log₂(sets) \| offset = log₂(block) |
| sets | size / (block × ways) |
| Typical | L1 32–64 KB ~4 cyc · L2 ~1 MB ~14 cyc · L3 ~40–70 cyc · DRAM ~100 ns |
| Misses | compulsory, capacity, conflict, (coherence) |
| Replacement | LRU, tree-PLRU, random, RRIP |
| Writes | WB + write-allocate (CPUs); WT + no-allocate + write buffer |
| Read/write flow | split address → index picks set → tag+valid check → hit: use offset · miss: evict victim (write back if dirty) → fetch whole block → install → then read/write; a write sets dirty (§2.6.1) |
| AMAT | HT₁ + MR₁(HT₂ + MR₂·MP₂); global MR₂ = MR₁·MR₂ |
| Optimisations | victim cache, non-blocking (MSHR), critical word first, prefetch, way prediction, banking |
| Code | stride-1, tiling, SoA, avoid pointer chasing, pad power-of-two strides, align to 64 B |
