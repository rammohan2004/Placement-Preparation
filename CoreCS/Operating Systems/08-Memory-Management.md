# 08. Memory Management

> **Asked at:** [AMZN] [DESHAW] paging vs segmentation, fragmentation, fit algorithms, page-table size numericals · [HFT] how `malloc` works, why it's avoided on the hot path, memory pools · [NVIDIA] [QCOM] kernel allocators (buddy, slab), physically contiguous memory for DMA · [TI] static allocation in embedded, MPU.
> **Prerequisites:** [02 Processes](02-Processes.md) (memory layout), [COA 07 Virtual Memory & TLB](../Computer%20Organization%20and%20Architecture/07-Virtual-Memory-and-TLB.md) (hardware side: page walk, TLB, multi-level tables, huge pages).

---

## 1. Intuition
Physical RAM is a **parking lot**. If cars (processes) must park in **one continuous stretch** (contiguous allocation), you soon get many small gaps that no big car fits into, even though the total free space is large (**external fragmentation**). **Paging** cuts every car into equal-size boxes that can go into **any** free slot, and keeps a **map** (page table) of where each box went. No gaps are wasted except a little inside each car's last box (**internal fragmentation**).

The OS's memory jobs:
1. Give each process the illusion of its own large, private memory (virtual memory).
2. Allocate and free physical frames.
3. Protect processes from each other.

---

## 2. Deep Dive

### 2.1 Logical vs physical addresses and address binding
- **Logical (virtual) address:** what the program uses. **Physical address:** the actual location in RAM. The **MMU** translates on every access.
- **When are addresses bound?**
  - compile time (absolute code: embedded/bare metal with a fixed load address);
  - load time (relocatable code fixed up by the loader);
  - **run time** (MMU translation, which is what all modern OSes do, so processes can move and share).
- Simplest MMU: **base + limit registers**. Physical = base + logical; trap if logical ≥ limit. It gives protection and relocation, but needs contiguous memory.

### 2.2 Contiguous allocation and fragmentation
Each process gets one continuous block. Free blocks are called **holes**.
| Fit policy | Rule | Notes |
|---|---|---|
| **First fit** | the first hole big enough | fast; good in practice |
| **Best fit** | the **smallest** hole big enough | leaves tiny useless slivers; must search all (or keep sorted) |
| **Worst fit** | the **largest** hole | leaves big leftovers; usually worst overall |
| Next fit | first fit, starting from where the last search ended | spreads allocations |
| | Internal fragmentation | External fragmentation |
|---|---|---|
| What | wasted space **inside** an allocated block | free space split into holes **between** blocks, none big enough |
| Cause | allocating in fixed units (pages, size classes) larger than requested | variable-size allocation and freeing |
| Example | asking for 5 KB with 4 KB pages ⇒ 8 KB used, 3 KB wasted | 3 holes of 10 KB free, but a 25 KB request fails |
| Fix | smaller units | **compaction** (move blocks; needs run-time binding), or **paging** |
**50 % rule** (first fit, statistical): for N allocated blocks, about 0.5 N blocks are lost to fragmentation ⇒ up to **1/3 of memory unusable**.

### 2.3 Paging (OS view)
- Virtual memory is divided into **pages**, physical memory into **frames** of the same size (4 KB typical; 2 MB / 1 GB huge pages).
- Virtual address = (page number, offset). The **page table** maps page number → frame number + permission bits.
- **No external fragmentation** (any free frame works); internal fragmentation ≤ 1 page per region (average ½ page).
- Each process has its own page table; the kernel switches the page-table base register on a context switch.
- **PTE bits:** valid/present, frame number, R/W/X permissions, user/supervisor, **dirty**, **accessed (referenced)**, caching attributes. The dirty and accessed bits are what page replacement uses (note 09).
- Large address spaces need **multi-level page tables** (x86-64: 4–5 levels), with only the used parts allocated. Translation is cached in the **TLB**. Hardware details: [COA 07](../Computer%20Organization%20and%20Architecture/07-Virtual-Memory-and-TLB.md).
- **Inverted page table:** one entry per **physical frame** (not per virtual page) + a hash lookup. Size ∝ RAM, not address space (old PowerPC, IA-64). Sharing pages is harder.
- **Page-size trade-off:**
  - bigger pages ⇒ smaller page tables, more TLB reach, fewer faults on sequential access, **but** more internal fragmentation and coarser protection/sharing;
  - smaller pages ⇒ the reverse.

### 2.4 Segmentation
Memory is divided into variable-size **logical units**: code, data, stack, heap. Address = (segment number, offset), translated with a segment table of (base, limit).
| | Paging | Segmentation |
|---|---|---|
| Unit size | fixed (page) | variable (segment) |
| Visible to programmer? | no | yes (logical units) |
| Fragmentation | internal (small) | **external** |
| Protection/sharing | per page | per logical unit (natural) |
| Today | **universal** | x86-64 keeps only a vestige (FS/GS registers for **thread-local storage**) |
**Segmentation with paging** (old x86-32): segments for logical structure, each segment paged. Modern OSes use a flat segment model and paging only.

### 2.5 Kernel memory allocators [NVIDIA][QCOM]
**Buddy allocator** (Linux page allocator): manages physical memory in blocks of 2ᵏ pages.
- Request n pages ⇒ round up to 2ᵏ. If there's no free block of that size, **split** a larger block in halves ("buddies") repeatedly.
- On free, if the buddy is also free ⇒ **merge** (coalesce) back up. The buddy's address is found by flipping one bit ⇒ fast.
- Pro: fast split/merge, limits external fragmentation. Con: internal fragmentation from rounding to a power of two (up to ~50 %).
- Gives **physically contiguous** memory, which is needed for DMA buffers on devices without an IOMMU (`kmalloc`, `dma_alloc_coherent`). `vmalloc` gives virtually contiguous memory made of scattered frames.

**Slab allocator** (Linux SLUB): caches of **pre-constructed objects of one size/type** (`task_struct`, inodes, network buffers), carved from pages.
- Allocation = pop from a per-CPU free list ⇒ very fast, no fragmentation within the cache, cache-friendly.
- `kmalloc` uses size-class slab caches (8, 16, 32, … bytes).

### 2.6 User-space heap: how `malloc` works [HFT]
```
malloc(n):
  small n:  find a free chunk in the right size-class "bin" (thread cache → arena bins)
            → split a bigger chunk if needed
            → if the heap is exhausted: grow it with brk()/sbrk() (move the program break up)
  large n (≥ ~128 KB in glibc, adjustable): mmap() a fresh region directly; free() → munmap()
free(p):   mark the chunk free, merge with free neighbours, put it in a bin
           (memory usually NOT returned to the OS, except mmap'd chunks or a trimmed top)
```
- Each chunk has a small **header** (size, flags) before the returned pointer ⇒ overhead + alignment (16 B on 64-bit).
- **Multithreading:** glibc uses multiple **arenas** + per-thread caches (tcache) to reduce lock contention. **jemalloc** (Facebook, FreeBSD) and **tcmalloc** (Google) use per-thread/per-CPU caches and size classes, and fragment less.
- **Physical memory is assigned lazily:** `malloc(1 GB)` returns quickly. Frames are allocated on **first touch** via page faults (note 09).
- **Why HFT avoids `malloc`/`new` on the hot path:**
  - unpredictable latency: lock contention, searching bins, occasionally `brk`/`mmap` syscalls and page faults;
  - fragmentation; cache misses on allocator metadata.
  Instead: **pre-allocate** at start-up, use **object pools / free lists / arena (bump) allocators**, `std::pmr` resources, and **touch + `mlock`** memory up front.

### 2.7 Stack vs heap
| | Stack | Heap |
|---|---|---|
| Allocation | automatic (move the stack pointer): ~free | `malloc`/`new`: tens of ns, sometimes µs |
| Lifetime | until the function returns | until `free`/`delete` |
| Size | small (8 MB default main thread; RTOS tasks: KBs) | large (limited by RAM + overcommit) |
| Fragmentation | none | possible |
| Errors | stack overflow (deep recursion, huge local arrays) ⇒ SIGSEGV via a guard page | leaks, double free, use-after-free, fragmentation |
| Tools | `ulimit -s` | Valgrind, AddressSanitizer, heaptrack |

### 2.8 Embedded memory [TI][QCOM]
- Many MCUs have **no MMU** (one physical address space). An **MPU** (memory protection unit) protects a few regions (e.g. stack guard, peripheral regions) but has no translation.
- **Static allocation** is preferred: allocate everything at start-up; no `malloc` after init (avoids fragmentation and non-determinism; MISRA rules).
- The linker script places sections: `.text`/`.rodata` in flash, `.data` copied flash → RAM at boot, `.bss` zeroed, stack/heap regions defined explicitly.

---

## 3. Worked Numericals

**N1. Fit algorithms (classic).** Holes (in order): 100, 500, 200, 300, 600 KB. Requests in order: 212, 417, 112, 426 KB. (A chosen hole is split; the leftover remains a hole in the same position.)
| Request | First fit | Best fit | Worst fit |
|---|---|---|---|
| 212 | 500 → leaves 288 | 300 → leaves 88 | 600 → leaves 388 |
| 417 | 600 → leaves 183 | 500 → leaves 83 | 500 → leaves 83 |
| 112 | 288 → leaves 176 | 200 → leaves 88 | 388 → leaves 276 |
| 426 | holes 100,176,200,300,183 ⇒ **fails (waits)** | 600 → leaves 174 ✓ | holes 100,83,200,300,276 ⇒ **fails** |
Only **best fit** places all four here. (In general first fit is about as good and faster.)

**N2. Single-level page-table size.** 32-bit virtual addresses, 4 KB pages, 4 B PTEs.
Offset = log₂(4096) = 12 bits ⇒ page number = 20 bits ⇒ 2²⁰ entries × 4 B = **4 MB per process**, even if the process uses 1 MB. That's why multi-level tables exist.
Two-level split 10 | 10 | 12: outer table 2¹⁰ × 4 B = 4 KB. A process touching 4 MB of contiguous memory needs 1 inner table (4 KB) ⇒ **8 KB** total instead of 4 MB.

**N3. 64-bit: why multiple levels.** 48-bit VA, 4 KB pages, 8 B PTEs, single level: 2³⁶ entries × 8 B = **512 GB** per process ⇒ impossible ⇒ 4 levels of 9 bits each (512 entries × 8 B = one 4 KB page per table).

**N4. Internal fragmentation with paging.** A process needs 72,766 B with 2 KB pages.
72,766 / 2048 = 35.53 ⇒ **36 pages**; allocated 36 × 2048 = 73,728 B ⇒ internal fragmentation = 73,728 − 72,766 = **962 B**.

**N5. Buddy allocator.** 1 MB block; requests A = 100 KB, B = 240 KB, C = 64 KB.
- A: round to 128 KB. Split 1 M → 512 + 512 → 256 + 256 → 128 + 128 ⇒ A gets 128 KB (internal frag 28 KB). Free: 128, 256, 512.
- B: round to 256 KB ⇒ take the free 256 ⇒ internal frag 16 KB. Free: 128, 512.
- C: 64 KB ⇒ split the free 128 → 64 + 64 ⇒ C gets 64 KB (no waste). Free: 64, 512.
Free A ⇒ its buddy (the 128 KB holding C plus a free 64 KB) isn't wholly free ⇒ no merge yet.

**Try it:** a system with 16-bit virtual addresses and 256-byte pages. How many pages? If PTEs are 2 B, how big is the page table? Which page and offset does address 0x3A7F map to?
<details><summary>Answer</summary>Offset = 8 bits ⇒ page bits = 8 ⇒ <b>256 pages</b>; table = 256 × 2 B = <b>512 B</b>. 0x3A7F ⇒ page <b>0x3A</b> (58), offset <b>0x7F</b> (127).</details>

---

## 4. Interview Questions

### Basic
- **Q: Logical vs physical address?** §2.1. The MMU translates at run time.
- **Q: Internal vs external fragmentation?** §2.2 table.
- **Q: Paging vs segmentation?** §2.4 table. Paging has no external fragmentation.
- **Q: First/best/worst fit?** §2.2, N1.

### Intermediate
- **Q: Why multi-level page tables?** A single-level table for large address spaces is huge (N2, N3); multi-level allocates only the used parts.
- **Q: What are the dirty and accessed bits for?** Dirty ⇒ the page must be written back before eviction. Accessed ⇒ approximates LRU for replacement (Clock, note 09).
- **Q: Stack vs heap?** §2.7.
- **Q: What happens in `malloc`? Is it a syscall?** §2.6. Usually no syscall: it uses bins and grows via `brk`/`mmap` only when needed.

### Advanced / killer follow-ups
- **Q: `malloc(1 GB)` succeeded, but the machine has 512 MB free. How?** Lazy allocation + overcommit: only virtual address space is reserved; frames come on first touch; if they run out, the OOM killer acts (note 09).
- **Q: Buddy vs slab: why does Linux use both?** Buddy = page-granularity physically contiguous blocks; slab = small fixed-size objects carved from those pages, fast per-CPU, no fragmentation per type.
- **Q [HFT]: Why is `new` banned in your hot path, and what do you use instead?** Unpredictable latency (locks, bin search, syscalls, page faults), fragmentation, cache misses ⇒ pre-allocated pools, arena/bump allocators, `std::pmr::monotonic_buffer_resource`, pre-faulted + `mlock`ed memory.
- **Q [NVIDIA][QCOM]: Why must DMA buffers sometimes be physically contiguous? How do you get them?** A device without an IOMMU uses physical addresses and can't follow page tables ⇒ `dma_alloc_coherent`/CMA, or scatter-gather lists. With an IOMMU, the device sees virtual-contiguous addresses.
- **Q: Where do FS/GS segments still matter on x86-64?** Thread-local storage: `fs:` points to the current thread's TLS block.

---

## 5. Common Traps
- Saying paging has **no** fragmentation (it has internal fragmentation).
- Forgetting the offset bits when computing the number of pages.
- Confusing virtually contiguous (`vmalloc`, `malloc`) with physically contiguous (`kmalloc`, DMA).
- Thinking `free()` returns memory to the OS immediately.
- "Best fit is always best". It creates many tiny holes; first fit is usually comparable and faster.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Binding | run time via MMU (base/limit → paging) |
| Fragmentation | internal (inside fixed units) vs external (holes between) |
| Fits | first (fast), best (tiny slivers), worst (usually worst) |
| Paging | fixed pages ↔ frames; page table; no external fragmentation; PTE: valid, frame, RWX, U/S, dirty, accessed |
| Table size | entries = 2^(VA bits − offset bits); × PTE size; multi-level for 64-bit |
| Segmentation | variable logical units; external fragmentation; only FS/GS (TLS) survives |
| Buddy | powers of two, split/merge buddies, physically contiguous |
| Slab | per-type object caches, per-CPU, fast |
| malloc | bins/arenas/tcache; `brk` small, `mmap` large (≥128 KB); lazy physical allocation |
| HFT | no malloc on hot path: pools, arenas, pre-fault + mlock |
| Embedded | no MMU (MPU), static allocation, linker script sections |
