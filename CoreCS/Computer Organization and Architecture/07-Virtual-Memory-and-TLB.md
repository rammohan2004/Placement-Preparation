# 07. Virtual Memory & TLB

> **Asked at:** All. [APPLE] [QCOM] [NVIDIA] VIPT caches, TLB design, page-table walks, 16 KB pages · [HFT] huge pages, TLB misses, page faults in the hot path, pre-faulting · [DESHAW] [AMZN] page-table size numericals, page fault handling (overlaps with OS).
> **Prerequisites:** [06 Cache](06-Cache-Memory.md).

---

## 1. Intuition
Every process believes it owns a huge, private, contiguous memory starting at address 0. The hardware (**MMU**) translates each **virtual address** to a **physical address** using per-process **page tables** maintained by the OS. Memory is managed in fixed-size **pages** (4 KB typical). Since translation would need extra memory accesses on *every* load, a small cache of translations — the **TLB** — makes it nearly free.

Virtual memory gives: **isolation & protection**, **illusion of more memory** (demand paging/swap), **no external fragmentation**, **sharing** (shared libraries, `mmap`), **copy-on-write** (`fork`).

---

## 2. Deep Dive

### 2.1 Paging basics
```
 Virtual address:   | Virtual Page Number (VPN) | Page offset (12 bits for 4 KB) |
                               │ page table lookup        │ (unchanged)
 Physical address:  | Physical Page Number (PPN)| Page offset                    |
```
**Page Table Entry (PTE)** contains: PPN, **valid/present**, **R/W/X permissions**, **user/supervisor**, **dirty**, **accessed (referenced)**, cacheability/memory type, global bit, (NX bit).

### 2.2 The page-table size problem → multi-level tables
Flat table for 32-bit VA, 4 KB pages, 4 B PTEs: 2²⁰ × 4 B = **4 MB per process** even if the process uses 1 MB. For 64-bit it's absurd.

**Hierarchical (radix-tree) tables** allocate only the parts in use.
x86-64 (4-level, 48-bit VA):
```
 47      39 38      30 29      21 20      12 11         0
 | PML4(9) | PDPT(9)  |  PD(9)   |  PT(9)   | offset(12) |
 CR3 ─► PML4 ─► PDPT ─► PD ─► PT ─► physical page
 Each table = 512 entries × 8 B = 4 KB = exactly one page.
 5-level paging (LA57) → 57-bit VA.  ARMv8: TTBR0/TTBR1, 4/16/64 KB granules.
```
Other designs: **inverted page table** (one entry per physical frame, hash lookup — PowerPC/IA-64), hashed page tables.

### 2.3 TLB — Translation Lookaside Buffer
A small (usually highly or fully associative) cache: `VPN (+ ASID) → PPN + permissions`.
| Typical (x86 big core, varies) | Entries | Latency |
|---|---|---|
| L1 DTLB | 64–96 (4 KB), plus 32 for 2 MB | ~0–1 cycle (parallel with L1 indexing) |
| L1 ITLB | 64–256 | |
| L2 STLB (unified) | 1.5–3 K | ~7–10 cycles |
| Page walk | — | 4 memory refs, tens to hundreds of cycles (walk caches help) |

- **TLB reach** = entries × page size. 64 × 4 KB = 256 KB only! ⇒ random access over a few MB thrashes the L1 TLB.
- **Page-walk caches / paging-structure caches** cache upper-level entries so walks skip levels; the walker's loads go through the data caches.
- Hardware-walked (x86, ARM) vs software-managed TLB (MIPS: TLB miss traps to OS).

### 2.4 Context switches: ASID / PCID
Without tags, a context switch must **flush the TLB** (new address space). With **ASIDs** (ARM) / **PCIDs** (x86), entries are tagged with the address-space ID ⇒ no flush. **Global** pages (kernel) are shared by all.
(Meltdown mitigation KPTI made PCIDs crucial for syscall performance.)

### 2.5 TLB shootdown (multicore)
When the OS changes/unmaps a PTE, other cores may have stale TLB entries ⇒ the OS sends **inter-processor interrupts (IPIs)** to make them invalidate ⇒ expensive (microseconds). [HFT] avoid `munmap`/`mprotect`/memory-returning allocators in the hot path; isolate cores.

### 2.6 Where does the cache sit relative to the TLB?
| Scheme | Index with | Tag with | Pros | Cons |
|---|---|---|---|---|
| PIPT | physical | physical | simple, no aliasing | TLB on critical path before cache |
| VIVT | virtual | virtual | fastest | **homonyms** (same VA, different processes) & **synonyms/aliases** (different VAs → same PA) ⇒ flushes, complexity |
| **VIPT** | virtual (page-offset bits only) | physical | TLB lookup **in parallel** with set indexing; no aliasing if index ⊆ offset | size limited |

**The VIPT constraint (very common question):**
```
 index + block-offset bits ≤ page-offset bits
 ⇔  cache size / associativity ≤ page size
```
- 4 KB pages, 32 KB L1 ⇒ ≥ 8-way. 48 KB ⇒ 12-way (Intel Golden Cove/Sunny Cove L1D: 48 KB 12-way).
- **Apple M-series uses 16 KB pages** ⇒ 128 KB L1D with 8 ways stays within VIPT limits — one reason Apple's L1 is so large.
- Breaking the constraint is possible but needs hardware to detect/handle synonyms.

### 2.7 Page fault handling (end-to-end)
1. Load → TLB miss → hardware page walk → PTE not present ⇒ **page fault exception** (precise).
2. OS handler: is the address valid (in a VMA)? If not → SIGSEGV.
3. Find a free frame (or evict a victim via page replacement — write back if dirty).
4. Load page from disk/swap or zero-fill (anonymous) or copy (copy-on-write).
5. Update PTE (present, PPN), possibly TLB invalidate.
6. Return from exception → **restart the faulting instruction**.
- **Minor fault:** page already in memory (e.g., first touch of zero page, page cache hit) ~ 1 µs. **Major fault:** disk I/O ~ ms (SSD ~ 10–100 µs).
- [HFT] **Pre-fault**: touch all memory at startup (`memset`, `MAP_POPULATE`), `mlock`/`mlockall` to prevent swapping, avoid lazy allocation in the hot path.

### 2.8 Huge pages [HFT][NVIDIA]
2 MB / 1 GB pages (x86), 64 KB / 2 MB / 1 GB (ARM).
- One TLB entry covers 512× more memory ⇒ far fewer TLB misses, shorter walks (3 or 2 levels).
- Linux: `hugetlbfs` (explicit, pre-reserved), **Transparent Huge Pages (THP)** — but THP compaction/khugepaged can cause latency spikes ⇒ HFT shops often prefer explicit huge pages.
- Cost: internal fragmentation, harder to allocate (need contiguous physical memory).
- GPUs use large pages (64 KB–2 MB) too; **Unified Memory** page faults on GPU are expensive.

### 2.9 Related OS mechanisms (quick)
- **Demand paging**, **copy-on-write** (fork shares pages read-only; first write faults & copies), **memory-mapped files** (`mmap`), **swap**, **thrashing** (working set > RAM).
- Page replacement: OPT, LRU, **Clock (second chance)** using the accessed bit, working-set model.
- Segmentation (x86-32 legacy; now FS/GS for thread-local storage).

### 2.10 Copy-on-write (CoW) and `fork()`, step by step
#### What `fork()` does
`fork()` creates a **child process that is a copy of the parent**: same code, same variables, same values. But from then on they have **separate memory**: a change made by one is not seen by the other.
```c
int x = 10;
pid_t pid = fork();          // returns 0 in the child, the child's PID in the parent (-1 on error)
if (pid == 0) {              // child
    x = 20;
    printf("child:  x = %d, &x = %p\n", x, (void*)&x);   // x = 20
} else {                     // parent
    wait(NULL);
    printf("parent: x = %d, &x = %p\n", x, (void*)&x);   // x = 10
}
// Both print the SAME virtual address &x, but different values:
// same virtual address, different physical pages (after the child's write).
```

#### The naive way, and why it's bad
Copy **all** of the parent's memory into new physical pages at `fork()` time:
- A process may use GBs ⇒ `fork()` would take a long time and double memory use.
- Very often the child immediately calls `exec()` (that's how a shell runs a command: `fork()` + `exec("ls")`), which **throws away** all that memory and loads a new program. The whole copy was wasted.

#### The copy-on-write idea
**Don't copy anything at `fork()`.** Let parent and child **share the same physical pages**, marked **read-only**. Copy a page **only when someone writes to it**, and only that one page.
*Analogy:* two students share one printed textbook. Nobody photocopies the whole book. Only when one wants to scribble on a page do they photocopy **that page** for themselves.

#### Step by step (one page holding `x`, at virtual page 5)
```
BEFORE fork():
  Parent page table:  VPN 5 → PPN 100  (read/write)

STEP 1: fork()
  - Kernel creates the child's page table by COPYING the parent's PTEs (not the data pages):
        Child page table:  VPN 5 → PPN 100
  - Marks the PTE READ-ONLY in BOTH page tables (W = 0), and remembers that the region is
    really writable but "shared copy-on-write".
  - Reference count of physical page 100 = 2 (two processes use it).
  - Flushes the parent's TLB entries for these pages (their permissions changed).
  → fork() is fast: only page tables were copied, no data.

STEP 2: child executes  x = 20  (a store to VPN 5)
  - MMU checks the PTE: write to a READ-ONLY page → PAGE FAULT (protection fault).
  - Kernel fault handler: "this region is writable and the page is CoW, so this isn't an error."
  - Allocates a new physical page 200 and copies 4 KB from page 100 → page 200.
  - Child PTE: VPN 5 → PPN 200, now READ/WRITE.   refcount(100) = 1.
  - Invalidates the child's stale TLB entry.
  - Returns from the fault → the store instruction is RESTARTED → writes 20 into page 200.

STEP 3: parent later writes to VPN 5
  - Still read-only → page fault.
  - Kernel sees refcount(100) = 1 (nobody else shares it) → no copy needed:
    just mark the parent's PTE read/write again and restart the store.

Pages that are never written (code, constants, untouched data) stay shared forever.
```
```
 After fork():                              After the child writes x:
 Parent: VPN 5 ─┐                           Parent: VPN 5 ──► PPN 100  (x = 10)
                ├──► PPN 100 (read-only,    Child:  VPN 5 ──► PPN 200  (x = 20, private copy)
 Child:  VPN 5 ─┘     refcount 2)
```

#### Who does what
| Hardware (MMU) provides | OS kernel implements |
|---|---|
| A **write-permission bit** in each PTE | Marking pages read-only at `fork()`, reference counts |
| A **page fault** when a write hits a read-only page | Deciding it's a CoW fault (not a real error), copying the page, fixing the PTE |
| **Precise exceptions**: the faulting store can be restarted | Flushing stale TLB entries |
This is why PTE permission bits and precise page faults matter so much: CoW, demand paging and `mmap` are all built on them.

#### Costs and benefits
| Benefit | Cost |
|---|---|
| `fork()` copies only page tables (fast) | Each **first write** to a shared page costs a page fault (~1–few µs) + a 4 KB copy |
| Memory is shared until written; fork + exec copies almost nothing | Page tables are still copied (noticeable for very large processes) |
| Read-only pages are shared forever | TLB flushes after changing permissions |
| | With **huge pages**, one CoW fault copies **2 MB**, not 4 KB |

**Real-world example (Redis) [AMZN][DESHAW][HFT]:** Redis saves a snapshot in the background with `fork()`: the child writes the data to disk while the parent keeps serving requests. Every page the parent modifies during the save gets copied, so memory use can grow towards 2× under heavy writes, and each first write to a page has a fault-latency spike. Redis recommends **disabling Transparent Huge Pages**, because with THP every CoW fault copies 2 MB.

#### Where else CoW appears
- `mmap(..., MAP_PRIVATE, ...)` of a file: you read the file's pages directly; your first write to a page gives you a private copy (the file isn't changed).
- **Zero page:** reading untouched (newly allocated) memory maps a shared all-zeros page; the first write allocates a real page.
- `vfork()` / `posix_spawn()`: avoid even the page-table copy when the child will `exec()` immediately.
- File systems (Btrfs, ZFS snapshots), and software types (Rust `Cow`, Swift arrays). Old copy-on-write `std::string` implementations were disallowed in C++11 (thread-safety and invalidation problems).

---

## 3. Worked Numericals

**N1. Flat page table size.** 32-bit VA, 4 KB pages, 4 B PTEs ⇒ 2³²/2¹² = 2²⁰ entries × 4 B = **4 MB** per process.

**N2. Levels needed.** 48-bit VA, 4 KB pages, 8 B PTE, each table must fit in one page.
Entries per table = 4096/8 = 512 ⇒ 9 bits per level. VPN = 48 − 12 = 36 bits ⇒ 36/9 = **4 levels**.

**N3. Effective memory access time with TLB.** TLB 1 ns, memory 100 ns, TLB hit rate 98%, 4-level table, no page-walk caches, no data caches.
- Hit: 1 + 100 = 101 ns. Miss: 1 + 4×100 + 100 = 501 ns.
- EAT = 0.98 × 101 + 0.02 × 501 = 98.98 + 10.02 = **109 ns**.

**N4. Page-fault rate budget.** Memory access 100 ns, page fault service 8 ms. For < 10% slowdown:
100 + p × 8×10⁶ < 110 ⇒ p < 1.25 × 10⁻⁶ ⇒ **fewer than 1 fault per 800,000 accesses**.

**N5. TLB reach.** 64-entry L1 DTLB, 1536-entry STLB.
- 4 KB pages: 256 KB / 6 MB.
- 2 MB pages: 128 MB / 3 GB (if the STLB holds 2 MB entries).
A 1 GB order-book / lookup table with random access: 4 KB pages ⇒ nearly every access misses the TLB; 1 GB pages ⇒ **1 entry**.

**N6. VIPT constraint.** 4 KB pages, 64 B lines. Max L1 size for a 4-way VIPT cache without aliasing = 4 KB × 4 = **16 KB**. To build a 64 KB VIPT L1 you need **16 ways** (or 16 KB pages with 4 ways).

**N7. Two-level table memory.** 32-bit VA, 10/10/12 split, 4 B PTEs. A process uses: code 4 MB at the bottom, heap 4 MB just above, stack 4 MB at the top.
- Each 2nd-level table maps 1024 × 4 KB = 4 MB. Needed: 3 second-level tables (if regions are 4 MB-aligned) + 1 top table ⇒ 4 × 4 KB = **16 KB** vs 4 MB flat.

---

## 4. Interview Questions

### Basic
- **Q: Why virtual memory?** Isolation/protection, abstraction of contiguous memory, demand paging, sharing, CoW.
- **Q: What's in a PTE?** (2.1.)
- **Q: What is a TLB and why is it needed?** Caches translations so each access doesn't need 4+ extra memory reads.
- **Q: What happens on a page fault?** (2.7.)

### Intermediate
- **Q: Why multi-level page tables?** Sparse address spaces: allocate only the subtrees that are used.
- **Q: What's TLB reach and how do you increase it?** Entries × page size; more entries, huge pages, multi-level TLBs.
- **Q: Why tag TLB entries with ASID/PCID?** Avoid flushing on every context switch.
- **Q: Minor vs major fault?** No disk I/O vs disk I/O.
- **Q [AMZN][DESHAW]: What is copy-on-write and why does `fork()` use it?** fork shares the parent's physical pages read-only instead of copying them; the first write to a page faults and copies just that page. Makes fork fast and saves memory, especially for fork + exec (§2.10).
- **Q: After `fork()`, parent and child print `&x` and get the same address. Is it the same memory?** Same **virtual** address; initially the same physical page (shared read-only), but after either writes, they're on different physical pages.

### Advanced / killer follow-ups
- **Q [APPLE][QCOM]: Explain VIPT and its size constraint. How does Apple get a 128 KB L1D?** (2.6.) 16 KB pages ⇒ 14 page-offset bits ⇒ 16 KB × 8 ways = 128 KB.
- **Q: What are synonyms (aliases) and homonyms? Which cache organisation suffers from each?** Synonym: two VAs → same PA (VIVT and VIPT beyond the constraint); homonym: same VA in two processes → different PAs (VIVT without ASIDs).
- **Q: What is a TLB shootdown and why is it expensive?** IPIs to all cores that may cache the mapping, each core interrupts and invalidates, initiator waits ⇒ µs, scales with cores.
- **Q [HFT]: Your latency spikes occasionally to 100 µs. VM-related suspects?** Page faults (lazy allocation, CoW after fork, swapped pages), THP compaction, TLB shootdowns from another thread's `munmap`, NUMA page migration (AutoNUMA). Fixes: pre-fault + `mlockall`, explicit huge pages, disable THP/AutoNUMA, avoid freeing memory to the OS in the hot path.
- **Q: Can the page-table walk itself hit in the data cache?** Yes — PTEs are ordinary memory; walkers read through L1/L2/L3. Page-walk caches also cache upper-level entries.
- **Q [NVIDIA]: How does GPU virtual memory differ?** GPUs have their own MMU/TLBs (per-SM L1 TLBs, shared L2 TLB), use larger pages, and Unified Memory can fault pages between CPU and GPU (expensive — prefetch with `cudaMemPrefetchAsync`).

---

## 5. Common Traps
- Forgetting that the page offset is not translated.
- Counting page-walk memory references wrong (4-level walk = 4 references, then the data access = 5).
- Mixing up "page size" constraint for VIPT (it's cache size / ways, not cache size).
- Saying TLB miss = page fault (a TLB miss is usually just a page walk; a fault is when the PTE is invalid).

---

## 6. Revision Summary
| | |
|---|---|
| VA → PA | VPN translated, offset unchanged |
| Flat table | 2^(VA − offset) × PTE size |
| x86-64 | 4 levels × 9 bits + 12 offset = 48 bits |
| TLB | ~64 L1 entries, ~1.5–3K L2; reach = entries × page size |
| EAT | h(t + m) + (1−h)(t + (L+1)m) |
| VIPT | cache size / ways ≤ page size |
| ASID/PCID | no flush on context switch |
| Huge pages | 2 MB / 1 GB → reach × 512; HFT favourite |
| Shootdown | IPIs, µs, avoid in hot path |
| Page fault | trap → find frame → load → update PTE → restart instruction |
| Copy-on-write | fork shares pages read-only (refcount); first write → fault → copy that page → make writable → restart store |
