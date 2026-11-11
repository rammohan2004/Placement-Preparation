# 09. Virtual Memory: Page Faults & Page Replacement

> **Asked at:** [AMZN] [DESHAW] demand paging, page-fault handling steps, FIFO/LRU/OPT numericals, Belady's anomaly, thrashing · [HFT] major vs minor faults, `mlockall`, pre-faulting, `mmap` · [NVIDIA] pinned (page-locked) memory, unified memory page faults · [APPLE] memory pressure, compression, no swap on iOS · [QCOM] Android low-memory killer.
> **Prerequisites:** [08 Memory Management](08-Memory-Management.md), [COA 07 VM & TLB](../Computer%20Organization%20and%20Architecture/07-Virtual-Memory-and-TLB.md) (page walk, TLB, copy-on-write §2.10).

---

## 1. Intuition
Your desk (RAM) is small; the library (disk/SSD) is huge. You keep only the books (pages) you're using on the desk. When you need a book that's not on the desk (**page fault**), you walk to the library to fetch it, which is slow. If the desk is full, you must send one book back (**page replacement**). Pick the book you won't need for the longest time (**OPT**), or approximately the one you haven't touched for longest (**LRU**). If your task needs more books than the desk holds, you spend all your time walking back and forth and do no work (**thrashing**).

---

## 2. Deep Dive

### 2.1 Demand paging
- A page is loaded into RAM **only when first accessed**, not at program start.
- PTE **valid/present bit = 0** ⇒ the access traps to the kernel = **page fault**.
- Benefits: faster start-up, less memory used, programs larger than RAM, cheap `fork` (CoW) and `malloc` (lazy).

### 2.1.1 What is in RAM: anonymous pages vs the page cache
> Read this first. §2.2, §2.4, §2.6 and §2.7 use the words *anonymous page*, *page cache*, *swap*, *clean* and *dirty*.

Every page that a process has in RAM is one of two kinds. What separates them is **whether a file on disk stands behind the page**.

| | **Anonymous page** | **File-backed page** (lives in the **page cache**) |
|---|---|---|
| What it holds | heap (`malloc`), stack, global variables after you write them | a 4 KB piece of a **file**: your program's own code, shared libraries (`libc.so`), files you `read()`/`write()`/`mmap` |
| Copy on disk? | **No.** It exists only in RAM, so it has no file name ("anonymous") | **Yes**: the file itself |
| How the kernel evicts it | first write it to **swap** (a disk area or file reserved for anonymous pages). **No swap ⇒ it can't be evicted** | **clean** (unchanged since read from disk): **just drop it** and re-read it from the file later. **dirty** (modified): write it back to the file first |
| Shared? | private to the process (except shared memory) | **one copy in RAM, shared** by every process using that file |

- **Dirty / clean:** the hardware sets the PTE's *dirty bit* when the page is written. Dirty means "RAM differs from disk", so evicting it costs an extra disk write. Clean pages are the cheapest to evict.
- **Page cache:** the kernel's collection of file pages kept in RAM, looked up by *(which file, which 4 KB page of the file)*. **Every file access goes through it.**

```
read(fd, buf, 4096) at file offset 8192   (= page 2 of the file)
        │
        ▼
kernel: is (this file, page 2) in the page cache?
   ├── YES (hit)  → copy 4 KB from that RAM frame into buf          ~1 µs
   └── NO  (miss) → take a free frame, read page 2 from disk into it  ~100 µs (NVMe)
                    add the frame to the page cache, then copy into buf
   (a 2nd read of the same page, by ANY process, is now a hit)
```
- **`write()`** copies your data into the page-cache page and marks it dirty, then returns. A kernel thread writes it to disk later (*writeback*, within ~30 s, or right away on `fsync`; see note 10).
- **`mmap` of a file** skips the copy: the kernel points your PTE straight at the page-cache frame (§2.6).
- **Page fault on a file page that is already in the cache ⇒ minor fault** (the kernel only fills in your PTE; no disk). Example: `libc.so` is used by every process, so its pages are almost always in the page cache already. When a new process first touches a libc page, it takes a ~1 µs minor fault, not a disk read.

> **"If the page is already in RAM, why is there a fault at all?"**
> Because *"the data is in RAM"* and *"**your** page table points to it"* are two separate facts. On every access the hardware (MMU + TLB) checks **only your process's page table**. The page cache is a kernel software structure the hardware knows nothing about.
> ```
> Page cache (kernel, shared):  (libc.so, page 3) ──► frame 0x5A        ← data IS in RAM
> Process A's page table:       VA 0x7f10_3000 ──► frame 0x5A, present = 1   (A touched it before)
> Process B's page table:       VA 0x7f88_3000 ──► empty,      present = 0   (B never touched it)
> ```
> B touches the address ⇒ present = 0 ⇒ **fault**. The handler finds (libc.so, page 3) in the page cache, writes B's PTE (frame 0x5A, present = 1), and the instruction re-runs. Nothing is loaded from disk, **only the mapping is created** ⇒ minor fault.
> Why B's PTE is empty:
> - (1) **Demand paging applies to page tables too.** `exec`/`mmap` only record the region (VMA, "addresses X–Y = libc.so") and fill **no** PTEs. Each PTE is filled on first touch.
> - (2) **CoW after `fork`.** The page is present but read-only, so the first write faults and the kernel copies it.
> - (3) **The kernel removed your mapping earlier, but the frame wasn't reused yet** (e.g. an anonymous page in the "swap cache"). The kernel just re-maps it.
>
> Why not fill all PTEs up front? Most pages are never touched (a program uses maybe 50 of libc's ~500 pages), so it would waste time and page-table memory. Linux compromises:
> - **fault-around:** one file fault also maps up to 16 neighbouring cached pages;
> - **`MAP_POPULATE`:** fills everything up front when you ask (HFT).

> **"How does the kernel know the page is already in RAM?"**
> The **kernel** keeps an index and the fault handler looks it up. The hardware doesn't know, and neither does your process.
>
> **Step 1: turn the faulting address into (file, page number).** The faulting virtual address is given to the kernel (x86: register CR2). The kernel finds your **VMA** (region) containing it. A file-backed VMA records *which file* and *from which file offset*:
> ```
> VMA: start 0x7f88_0000, file = libc.so, file offset 0      Fault at VA 0x7f88_3010
> page index = (0x7f88_3010 − 0x7f88_0000) / 4096 + 0 = 3   ⇒ "libc.so, page 3"
> ```
> **Step 2: look it up in that file's page-cache index.** Every file has one kernel object, the **inode**, shared by all processes that open or map it (note 10). Each inode holds a table *page number → RAM frame* (Linux: an *xarray*, a radix tree, in the inode's `address_space`).
> ```
> inode(libc.so) → index:  page 0 → frame 0x51 | page 1 → frame 0x77 | page 3 → frame 0x5A | page 4 → absent
> ```
> - **Found** ⇒ write your PTE → frame 0x5A ⇒ **minor fault**.
> - **Absent** ⇒ allocate a frame, **insert it into the index first** (marked locked / not filled), start the disk read, sleep ⇒ **major fault**. A second process faulting on the same page meanwhile finds the locked entry and waits for *that* read, so the page is never read twice.
> - Every path (`read()`, `mmap` faults, `exec` loading code) uses this same index. That's why a page loaded by anyone is visible to everyone.
>
> **Step 3: anonymous pages (no file) use the old PTE bits.** When present = 0 the hardware ignores the other PTE bits, so the kernel stores its own information there:
>
> | PTE at fault time | Meaning | Kernel action |
> |---|---|---|
> | all zero | never mapped | file VMA → page-cache lookup (above); anonymous VMA → zeroed frame (minor) |
> | present = 0, rest = **swap entry** (swap slot number) | page was swapped out | look in the **swap cache** (index of swap slots still in RAM): hit → re-map (minor), miss → read swap (major) |
> | present = 1, read-only, you wrote | copy-on-write | copy the page (minor) |
>
> A program can ask too: **`mincore(addr, len, vec)`** reports which pages of a mapping are resident (the `vmtouch` tool uses it to check whether a file is "warm").

> **"If the kernel has the page-cache index, why do we need page tables?"** They answer different questions for different users:
>
> | | Page-cache index (per **file**) | Page table (per **process**) |
> |---|---|---|
> | Question | "Is page 3 of libc.so in RAM? Where?" | "What does VA 0x7f88_3010 mean **in process B**?" |
> | Key → value | (file, page no.) → frame | (process, VA) → frame **+ permissions** (R/W/X, user, accessed, dirty) |
> | Read by | kernel software, only on faults / `read()` | **MMU hardware**, on every TLB miss |
> | Covers | file pages only | every page (heap, stack, files, shared memory) |
>
> Why the page cache can't replace page tables:
> 1. **The CPU issues virtual addresses.** The needed translation is VA → frame, and the same file page sits at **different VAs** in different processes (ASLR).
> 2. **Heap and stack pages have no file**, so they're not in the page cache at all.
> 3. **Permissions are per mapping.** The same page can be read-only in A and writable in B, or privately copied after CoW. A page-cache entry exists once per file page.
> 4. **Isolation.** The page table *is* the list of what a process may touch; the page cache is global.
> 5. **Speed.** The MMU walks page tables (fixed hardware format, x86-64: 4 levels × 512 entries) in ~20–50 ns with no software. Hardware can't walk the kernel's software structure. Trapping to the kernel on every TLB miss: 10⁶ misses/s × 1 µs = **100 % of the CPU**, vs 10⁶ × 30 ns = **3 %** for a hardware walk.
>
> ```
>  page cache: (libc.so, page 3) → 0x5A ──consulted ONCE, at the fault──┐
>                                                                       ▼
>  A's page table: VA 0x7f10_3000 → 0x5A R-X  ┐
>  B's page table: VA 0x7f88_3000 → 0x5A R-X  ├─ read by the MMU on EVERY access
>  B's page table: VA 0x5600_2000 → 0x91 RW-  ┘  (0x91 = heap: not in the page cache)
> ```
> **The page fault is the bridge:** it copies "where in RAM" from the page cache into *your* page table once, and from then on the hardware translates alone. A third index, the **reverse map** (frame → every PTE pointing at it), lets the kernel clear A's and B's PTEs when it evicts frame 0x5A.
- Linux uses almost **all otherwise-unused RAM as page cache** and gives it back instantly when programs need memory, because clean pages are simply dropped. That's why `free -h` shows little "free" memory but large "buff/cache" and "available". This is normal, not a memory leak.
- Analogy: in the desk/library picture (§1), the page cache is a **shared shelf of photocopies** in the room. Anyone who fetched a library book leaves a copy there, and the next person who needs it takes it from the shelf instead of walking to the library.

### 2.2 Page-fault handling, step by step (OS view)
```
1.  CPU accesses virtual address V → TLB miss → page walk → PTE present = 0 → PAGE FAULT (exception)
2.  CPU saves state and jumps to the kernel's page-fault handler (faulting address in CR2 on x86)
3.  Kernel: is V inside a valid region (VMA) of this process, with the right permission?
       NO  → send SIGSEGV ("segmentation fault") → usually kills the process
       YES → continue
4.  Find a free physical frame. None free? → run PAGE REPLACEMENT: pick a victim frame;
       if the victim is DIRTY → write it to swap/file first (slow); update the victim's PTE (present = 0)
5.  Fill the frame:
       anonymous page touched for the first time → zero-fill (minor fault)
       file page already in the page cache (§2.1.1: shared lib, file) → just point the PTE at it (minor fault)
       page on disk/swap → issue the read; the process BLOCKS (state: waiting); another process runs
6.  I/O completes (interrupt) → update the PTE: frame number, present = 1
7.  Process becomes READY; when it runs, the faulting instruction is RE-EXECUTED and now succeeds
```
| Fault type | Meaning | Cost (typical) |
|---|---|---|
| **Minor (soft)** | no disk I/O: zero-fill, page already in the page cache, CoW copy | **~0.5–2 µs** |
| **Major (hard)** | must read from disk/swap | **~50–150 µs (NVMe SSD)**, **~5–10 ms (HDD)** |
| **Invalid** | address not mapped / permission violation | SIGSEGV |
See them: `/usr/bin/time -v ./prog`, `perf stat -e page-faults,major-faults`, `ps -o min_flt,maj_flt`.

### 2.3 Effective access time (EAT)
EAT = (1 − p) × memory access time + p × page-fault service time, where p = page-fault rate.
Since fault time (ms) ≫ memory time (ns), even a tiny p dominates (N1).

### 2.4 Page-replacement algorithms
| Algorithm | Victim | Notes |
|---|---|---|
| **FIFO** | the page loaded **earliest** | simple; can evict hot pages; suffers **Belady's anomaly** |
| **OPT / MIN (Belady)** | the page **not used for the longest time in the future** | **optimal**, but needs the future ⇒ only a benchmark |
| **LRU** | the page **least recently used** | good approximation of OPT (past predicts future); exact LRU is too costly in hardware (timestamps or a list update on **every** access) |
| **Clock (second chance)** | circular scan: if accessed bit = 1 → clear it and skip; if 0 → evict | **practical LRU approximation** using the hardware accessed bit |
| Enhanced clock (NRU) | prefer (accessed, dirty) = (0,0) > (0,1) > (1,0) > (1,1) | avoids writing dirty pages back when possible |
| LFU / MFU | least / most frequently used | rarely used alone |
**Linux:** two LRU lists per memory zone/cgroup, **active** and **inactive** (a page is promoted on a second access), with Clock-like scanning; newer kernels use **multi-gen LRU (MGLRU)**. Clean page-cache pages are the cheapest to drop; dirty or anonymous pages need writeback/swap.

**Belady's anomaly:** with FIFO, giving **more frames can cause more faults** (N3). LRU and OPT are **stack algorithms**: the pages in memory with n frames are always a subset of those with n + 1 frames ⇒ they never show the anomaly.

### 2.5 Frame allocation and thrashing
- **Equal** allocation (m frames / n processes) vs **proportional** (by process size): aᵢ = sᵢ / S × m.
- **Global replacement** (take a frame from any process: Linux default) vs **local** (only from yourself).
- **Thrashing:** the processes' combined working sets don't fit in RAM ⇒ constant page faults ⇒ the CPU waits on I/O ⇒ utilisation drops ⇒ (classic batch OS) the scheduler adds *more* processes to raise utilisation ⇒ even worse.
```
CPU
util │      ╭──╮
     │    ╭─╯  ╰╮
     │  ╭─╯     ╰─╮
     │╭─╯         ╰──╮ ← thrashing
     └──────────────────► degree of multiprogramming
```
- **Working set** W(t, Δ) = the set of pages referenced in the last Δ references. If Σ WSSᵢ > total frames ⇒ thrashing ⇒ **suspend/swap out** a process.
- **Page-fault frequency (PFF):** if a process's fault rate is above the upper bound, give it frames; below the lower bound, take frames away.
- Today: the **OOM killer** (Linux) or low-memory killer / jetsam (Android / iOS) kills processes instead.

### 2.6 mmap: memory-mapped files and anonymous memory
```c
int fd = open("data.bin", O_RDONLY);
char *p = mmap(NULL, len, PROT_READ, MAP_PRIVATE, fd, 0);  // file now appears as memory
sum += p[1000000];        // first touch → page fault → kernel reads that page from the page cache/disk
```
| | `MAP_SHARED` | `MAP_PRIVATE` |
|---|---|---|
| File-backed | writes go to the file (via the page cache); visible to other mappers | writes are **CoW**: private copy, file unchanged |
| Anonymous (`MAP_ANONYMOUS`) | shared memory between parent/child after fork | ordinary private memory (what `malloc` uses for big blocks) |
- **Why mmap?** No `read()` copy into a user buffer (pages map the page cache directly), lazy loading, easy sharing between processes (IPC, note 12). Shared libraries are mmapped.
- **Costs:** a page fault per 4 KB page on first touch, TLB pressure. Fixes: `MAP_POPULATE` (pre-fault), `madvise(MADV_SEQUENTIAL / WILLNEED)`, huge pages.

### 2.7 Locking (pinning) memory and avoiding faults [HFT][NVIDIA]

#### 2.7.1 What "pinning" a page means
Normally a page of your process in RAM is only **borrowed**. When RAM runs short, page replacement (§2.4) may take its frame and give it to someone else:
```
Your page P sits in frame 0x5A000, PTE present = 1
   │  RAM runs short → replacement picks P as the victim
   │     anonymous page  → written to swap            (§2.1.1)
   │     clean file page → simply dropped (e.g. your own CODE pages)
   ▼
PTE present = 0, frame now used by someone else
   │  ... later your code touches P
   ▼
MAJOR fault → read P back from disk → your thread waits ~100 µs (SSD) / ~10 ms (HDD)
```
**Pinning (= locking) a page** means telling the kernel: *"never take this page out of RAM."* Page replacement skips it, so the page is always present, and touching it **never causes a major fault**.
Analogy: a **"do not return" sticker** on a book on your desk. Even when the desk is full, the librarian must pick some other book to send back.

There are two strengths of pinning. Don't mix them up:
| | **`mlock` / `mlockall`** (any user program) | **DMA pin** (drivers: `pin_user_pages`; CUDA `cudaHostAlloc`) |
|---|---|---|
| Promise | page **stays in RAM** (never swapped or dropped) | page stays in RAM **and keeps the same physical frame** |
| Purpose | latency: no major faults | correctness: a device writes to that **physical** address |
| May the kernel *move* it to another frame? | yes. Linux can *migrate* it (copy it to a new frame and update the PTE) to defragment RAM. Your virtual address is unchanged; this is rare | **no** |

#### 2.7.2 `mlock` / `mlockall`
```c
#include <sys/mman.h>
mlockall(MCL_CURRENT | MCL_FUTURE);   // lock everything, now and later
mlock(buf, len);                      // or lock just one range
```
- **`MCL_CURRENT`** locks every page mapped **now**: code, globals, heap, the stack so far, shared libraries. Linux also **loads them all into RAM during the call**.
- **`MCL_FUTURE`** also locks anything mapped **later** (future `malloc`/`mmap`). Those pages are loaded when the mapping is created.
- **How the kernel does it:** the region is marked locked (`VM_LOCKED`), and its pages go on an *unevictable* list that the replacement scanner never looks at.
- **Limits:**
  - An ordinary user may lock only `RLIMIT_MEMLOCK` bytes (`ulimit -l`; often 8 MB, or 64 KB on older systems). Locking more needs root, `CAP_IPC_LOCK`, or a raised limit.
  - Locked RAM is unavailable to everyone else. Lock too much and other processes swap or get OOM-killed.
  - A locked process can still be killed by the OOM killer.
- **Turning swap off does NOT replace `mlockall`.** Without swap, anonymous pages can't be evicted. But **file-backed pages (your program's code, libc, mmapped files) can still be dropped and re-read**, which gives major faults on code paths that ran once and then sat idle.
- Same idea on other OSes: Windows `VirtualLock`, macOS `mlock`.

#### 2.7.3 Why HFT code still pre-touches memory after `mlockall`
`mlockall` removes **major** faults and loads the pages that are already mapped. Some **minor** faults can still happen *during* trading:
1. **Stack depth.** The stack grows on demand. Stack pages deeper than the deepest point reached so far aren't mapped yet, so the first deep call chain faults. Fix: at start-up, call a function with a large local array and write to it.
2. **Later allocations.** With `MCL_FUTURE`, a `malloc` during trading maps and loads new pages **inside that `malloc` call**, which is a latency spike. Fix: allocate every pool at start-up and never call `malloc` on the hot path.
3. **Insurance.** Writing one byte per page of every buffer guarantees each page has a real private frame, even for memory you forgot to lock or on a system where locking isn't allowed.
```c
static char pool[64 << 20];                       // 64 MB pre-allocated order pool
void prefault_at_startup(void) {
    mlockall(MCL_CURRENT | MCL_FUTURE);
    for (size_t i = 0; i < sizeof pool; i += 4096) pool[i] = 0;    // 1 write per 4 KB page
    volatile char stack_probe[1 << 20];                           // grow the stack by 1 MB now
    for (size_t i = 0; i < sizeof stack_probe; i += 4096) stack_probe[i] = 0;
}
```
Name-only (beyond typical interview depth): low-latency boxes also set `vm.compact_unevictable_allowed = 0` (stop migrating locked pages) and turn off automatic NUMA balancing, which deliberately causes "hinting" minor faults.

#### 2.7.4 [NVIDIA] Pinned (page-locked) host memory for GPU transfers
- **Why DMA needs a pin.** In **DMA**, a device (GPU, NIC, SSD) copies data to or from RAM by itself, using **physical** addresses, without the CPU ([COA 13](../Computer%20Organization%20and%20Architecture/13-IO-Interrupts-DMA.md)). The device does not look at your page table.
  - If the OS swapped out or moved the page in the middle of a transfer, the GPU would write into a frame that now belongs to someone else, silently corrupting memory.
  - So the frame must be pinned **at a fixed physical address** for the whole transfer.
```
Pageable buffer:  your buf ──CPU memcpy──► driver's pinned staging buffer ──DMA──► GPU
Pinned buffer:    your buf ───────────────────────── DMA ─────────────────────────► GPU
```
- `cudaMallocHost`/`cudaHostAlloc` give you pinned memory directly. With pageable memory, `cudaMemcpy` must first copy into the internal pinned staging buffer, which is slower and can't be asynchronous. Pinned memory ⇒ faster transfers + `cudaMemcpyAsync` overlap with kernels.
- Too much pinned memory starves the OS: those frames can never be reclaimed.
- **[NVIDIA] Unified Memory** (`cudaMallocManaged`): pages migrate between CPU and GPU **on page fault** (Pascal+). Convenient, but faults are expensive ⇒ use `cudaMemPrefetchAsync`.

### 2.8 Overcommit, swap, OOM
- Linux **overcommits** by default: `malloc`/`mmap` succeed beyond RAM + swap, betting that not all memory gets touched (`vm.overcommit_memory` 0 = heuristic, 1 = always, 2 = strict).
- When memory truly runs out, the **OOM killer** picks a victim by `oom_score` (mostly by memory size; adjust with `oom_score_adj`). In containers, the **cgroup memory limit** triggers OOM within that cgroup (note 15).
- **Swap:** anonymous pages evicted to disk. `vm.swappiness` sets the preference between dropping page cache and swapping anonymous memory. Latency-critical servers often disable swap or lock memory.
- [APPLE] macOS uses swap + **memory compression** (compress inactive pages in RAM before swapping). iOS traditionally has **no swap to disk** (flash wear, latency): it compresses memory and **jetsam** kills background apps under pressure. [QCOM] Android: zRAM (compressed RAM swap) + the low-memory killer (lmkd).

---

## 3. Worked Numericals

**N1. EAT and the tolerable fault rate.** Memory access 100 ns, page-fault service 8 ms.
EAT = (1 − p) × 100 + p × 8,000,000 ns ≈ 100 + 7,999,900 p ns.
- p = 1/1000 ⇒ EAT ≈ 100 + 8000 = **8.1 µs** ⇒ **81× slower**.
- For ≤ 10 % slowdown: 100 + 7,999,900 p ≤ 110 ⇒ p ≤ 1.25 × 10⁻⁶ ⇒ at most **1 fault per ~800,000 accesses**.

**N2. FIFO vs LRU vs OPT, 3 frames.** Reference string: 1 2 3 4 1 2 5 1 2 3 4 5 (F = fault, frames shown after the access).

FIFO:
| Ref | 1 | 2 | 3 | 4 | 1 | 2 | 5 | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Frames | 1 | 1 2 | 1 2 3 | 4 2 3 | 4 1 3 | 4 1 2 | 5 1 2 | 5 1 2 | 5 1 2 | 5 3 2 | 5 3 4 | 5 3 4 |
| F? | F | F | F | F | F | F | F | | | F | F | |
FIFO = **9 faults**.

LRU (evict the least recently used):
| Ref | 1 | 2 | 3 | 4 | 1 | 2 | 5 | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Frames | 1 | 1 2 | 1 2 3 | 4 2 3 | 4 1 3 | 4 1 2 | 5 1 2 | 5 1 2 | 5 1 2 | 3 1 2 | 3 4 2 | 3 4 5 |
| F? | F | F | F | F | F | F | F | | | F | F | F |
LRU = **10 faults**. (At "3": recency 5 < 1 < 2 ⇒ evict 5. At "4": evict 1. At "5": evict 2.)

OPT (evict the page used farthest in the future):
| Ref | 1 | 2 | 3 | 4 | 1 | 2 | 5 | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Frames | 1 | 1 2 | 1 2 3 | 1 2 4 | 1 2 4 | 1 2 4 | 1 2 5 | 1 2 5 | 1 2 5 | 3 2 5 | 4 2 5 | 4 2 5 |
| F? | F | F | F | F | | | F | | | F | F | |
OPT = **7 faults**. (At "4": next uses 1 → pos 5, 2 → 6, 3 → 10 ⇒ evict 3. At "5": 4 is next used at 11 ⇒ evict 4. At "3": 1 and 2 are never used again ⇒ evict either.)

**N3. Belady's anomaly.** Same string, FIFO with **4** frames:
1 2 3 4 (4 faults) → 1 hit, 2 hit → 5 F (evict 1) → 1 F (evict 2) → 2 F (evict 3) → 3 F (evict 4) → 4 F (evict 5) → 5 F (evict 1) ⇒ **10 faults** > 9 with 3 frames. More memory, more faults!
LRU with 4 frames on the same string = 8 faults (< 10): no anomaly.

**N4. Classic 20-reference string, 3 frames** (results to remember):
7 0 1 2 0 3 0 4 2 3 0 3 2 1 2 0 1 7 0 1 ⇒ **FIFO 15, LRU 12, OPT 9**. (Do the trace yourself as practice.)

**N5. Working set.** References: … 2 6 1 5 7 7 7 7 5 1 | (t₁) … with Δ = 10 at time t₁: last 10 refs = {2, 6, 1, 5, 7} ⇒ **WSS = 5**. If 4 processes have WSS 5, 12, 20, 8 = 45 and only 40 frames exist ⇒ Σ > frames ⇒ thrashing risk ⇒ suspend one.

**N6. Proportional allocation.** 62 free frames; process sizes 10 and 127 pages.
a₁ = 10/137 × 62 ≈ **4**, a₂ = 127/137 × 62 ≈ **57**.

**N7. [HFT] Cost of first-touch faults.** A strategy allocates a 1 GB order book at start-up and touches it during trading. 4 KB pages ⇒ 262,144 minor faults × 1 µs ≈ **0.26 s** of stalls scattered across trading, each a µs-scale spike. With 2 MB huge pages ⇒ 512 faults. Pre-touch at start-up ⇒ **0** faults during trading.

**N8. [HFT] Why `mlockall` even with swap disabled.** A trading process has 200 MB of code + data and is **not** locked; swap is off. Overnight, a log-compression job reads a 30 GB file. Those file pages fill the page cache, and replacement evicts least-recently-used pages, including **2,000** clean code pages of the trading process (code that ran once and then sat idle).
- Each page is file-backed and clean ⇒ simply dropped ⇒ next morning, the first execution of each page = **major fault** ≈ 100 µs (NVMe).
- Total stall = 2,000 × 100 µs = **0.2 s**, spread over the first trades. Each spike is 100 µs, which is **20×** a 5 µs tick-to-trade budget.
- With `mlockall(MCL_CURRENT|MCL_FUTURE)`: the pages are unevictable ⇒ **0** major faults.

**N9. Page cache: sharing and re-reads.**
- (a) 500 processes all use `libc.so` (~2 MB of code). The code pages sit **once** in the page cache, and every process's PTEs point at the same frames ⇒ **2 MB** of RAM, not 500 × 2 = 1000 MB. A new process's first touch of a libc page = a **minor** fault.
- (b) `cat` a 4 GB file twice on a machine with 16 GB RAM, NVMe at 2 GB/s, memory copy ≈ 10 GB/s. 1st run: 4 / 2 = **2 s** (disk). 2nd run: every page is a page-cache hit ⇒ 4 / 10 = **0.4 s**. Benchmarks must state "cold" or "warm" cache.

**Try it:** 4 frames, reference string 7 0 1 2 0 3 0 4 2 3 0 3 2. Count LRU faults.
<details><summary>Answer</summary>7F 0F 1F 2F (frames 7 0 1 2), 0 hit, 3F evict 7 → 3 0 1 2, 0 hit, 4F evict LRU = 1 → 3 0 4 2, 2 hit, 3 hit, 0 hit, 3 hit, 2 hit ⇒ <b>6 faults</b>.</details>

---

## 4. Interview Questions

### Basic
- **Q: What is a page fault? Is it an error?** An exception when the accessed page isn't present (or the permission is wrong). Usually *not* an error: demand paging, CoW, lazy allocation. Only invalid addresses become SIGSEGV.
- **Q: Steps to handle a page fault?** §2.2.
- **Q: FIFO vs LRU vs OPT?** §2.4. OPT is optimal but needs the future; LRU approximates it; FIFO is simplest but has Belady's anomaly.
- **Q: What is thrashing?** §2.5.

### Intermediate
- **Q: What is Belady's anomaly? Which algorithms are immune?** More frames ⇒ more faults with FIFO. Stack algorithms (LRU, OPT) are immune.
- **Q: How does the OS approximate LRU?** Clock / second chance using the hardware accessed bit; Linux active/inactive lists / MGLRU.
- **Q: Minor vs major page fault?** §2.2 table.
- **Q: How does mmap work, and why is it sometimes faster than read()?** §2.6: maps page-cache pages directly, no extra copy, lazy.
- **Q: What is the page cache?** RAM frames holding pieces of files, looked up by (file, page). Every `read`/`write`/`mmap` goes through it. A hit avoids the disk; writes are buffered as dirty pages and written back later. Linux fills spare RAM with it and drops clean pages instantly when programs need memory. *Follow-up:* "`free` shows almost no free memory. Problem?" No: look at "available", which includes droppable cache.
- **Q: The page is already in RAM (page cache). Why does my process still take a page fault?** The MMU checks only *your* page table, and your PTE is empty: `mmap`/`exec` fill PTEs lazily on first touch. The kernel finds the page in the cache and just writes your PTE ⇒ minor fault, no disk. *Follow-up:* "How do you avoid even those?" `MAP_POPULATE` / `mlockall` / pre-touch at start-up; fault-around already batches 16 pages per fault.
- **Q: On a page fault, how does the kernel know whether the page is already in RAM?** Faulting address → your VMA → (file, page index). It looks that up in the file's page-cache index (per-inode xarray): found ⇒ map it (minor); absent ⇒ insert a locked entry, read from disk (major). Anonymous pages: the non-present PTE holds a swap entry, so it checks the swap cache, then swap. *Follow-up:* "Two processes fault on the same uncached page at once?" The first inserts a locked page and starts the I/O; the second finds it and waits, giving one disk read.
- **Q: The kernel already indexes file pages in the page cache. Why do we still need page tables?** Different key and different reader. The page cache maps (file, page) → frame for the kernel, only on faults, and only for file pages. The page table maps (process, VA) → frame + per-process permissions, for the MMU hardware on every TLB miss, and covers heap/stack too. It also provides isolation, and a hardware walk (~30 ns) beats trapping to the kernel (~1 µs) per miss. The fault copies the page-cache answer into the page table once. *Follow-up:* "When a page-cache page is evicted, how are all the PTEs mapping it found?" The reverse map (rmap): frame → mapping PTEs.
- **Q: Anonymous vs file-backed page: what happens to each when it's evicted?** Anonymous (heap/stack) has no disk copy ⇒ it must be written to swap, and can't be evicted without swap. File-backed: clean ⇒ just dropped and re-read from the file later; dirty ⇒ written back to the file first.

### Advanced / killer follow-ups
- **Q [HFT]: How do you guarantee no page faults during trading?** `mlockall(MCL_CURRENT|MCL_FUTURE)`, pre-allocate and pre-touch all memory, huge pages, no `malloc` on the hot path, avoid touching new stack depth (pre-fault the stack), no file mmaps that can be evicted.
- **Q [HFT]: What does "pinning" a page mean, and what does `mlock` guarantee?** The page is never swapped out or dropped, so no major faults (§2.7.1). It does **not** guarantee a fixed physical address: Linux may migrate it to another frame. Fixed physical frames are a *DMA pin*, done by drivers (`pin_user_pages`, `cudaHostAlloc`).
- **Q [HFT]: Swap is disabled. Can my process still take major faults?** Yes. File-backed pages (its own code, shared libraries, mmapped files) can be dropped under memory pressure and re-read from disk (N8). Hence `mlockall`.
- **Q [HFT]: Why pre-touch memory after `mlockall`?** The stack grows on demand (deeper pages fault on first use), and allocations made later fault inside `malloc`. So pre-grow the stack, pre-allocate and touch every pool at start-up, and never allocate on the hot path (§2.7.3).
- **Q [NVIDIA]: Why is `cudaMemcpy` from pinned memory faster than from pageable memory?** §2.7.4: DMA needs physical pages that can't move; pageable memory requires an extra staging copy and can't be asynchronous. *Follow-up:* "Why not pin everything?" Pinned frames can never be reclaimed, so the OS runs short of memory.
- **Q: `malloc` returned non-NULL but the program was killed later. Why?** Overcommit: memory was only reserved virtually; at first touch there were no frames ⇒ the OOM killer.
- **Q: Why might reading a file twice be much faster the second time?** The page cache: the second read hits RAM (minor faults or a plain memory copy). Benchmark gotcha: drop caches (`echo 3 > /proc/sys/vm/drop_caches`) for cold measurements.
- **Q [APPLE]: How does iOS handle memory pressure without swap?** Compressed memory, memory-pressure notifications to apps, and jetsam terminating background/idle apps by priority.
- **Q: The page-fault handler itself must not page-fault. How is that ensured?** Kernel code and core data structures are never paged out (Linux kernel memory isn't swappable).

---

## 5. Common Traps
- Saying a page fault = segmentation fault.
- Forgetting that the faulting instruction is **restarted**, not skipped.
- In OPT traces, evicting based on the **past**. Look at the **future**.
- Thinking LRU is used exactly by real OSes. They use approximations.
- Ignoring the dirty bit: evicting a dirty page costs an extra write.
- Believing more RAM always reduces faults for every algorithm (Belady's anomaly for FIFO).
- "Swap off ⇒ no major faults." Wrong: clean file-backed pages (code, libs) are still dropped and re-read.
- "`mlock` ⇒ the physical address never changes." Wrong: it only guarantees the page stays in RAM. A fixed frame for DMA needs a driver pin.
- "Low free memory in `free -h` = a leak." Usually it's just page cache; check "available".

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Demand paging | load on first access; present = 0 ⇒ fault |
| Anonymous vs file-backed | heap/stack (no disk copy → swap) vs file pages (clean → drop, dirty → write back) |
| Page cache | file pages in RAM by (file, page); all read/write/mmap go through it; shared; hit ⇒ minor fault / no disk |
| In RAM but still a fault? | MMU checks only *your* PTE; PTEs are filled lazily ⇒ minor fault = "create the mapping", not "load the data" |
| How the kernel knows it's in RAM | VA → VMA → (file, page idx) → inode's page-cache index (xarray). Anonymous: PTE swap entry → swap cache. User check: `mincore` |
| Page cache vs page table | (file, page) → frame, kernel, on faults, file pages only **vs** (process, VA) → frame + perms, MMU, every TLB miss, all pages. Fault = bridge; rmap = frame → PTEs |
| Fault steps | trap → valid VMA? (else SIGSEGV) → frame (evict, write back if dirty) → fill → PTE → restart instruction |
| Minor / major | ~1 µs (no I/O) / ~100 µs SSD, ~10 ms HDD |
| EAT | (1−p)·mem + p·fault; p must be ~10⁻⁶ |
| FIFO / OPT / LRU | earliest / farthest future / least recent; OPT optimal |
| Clock | accessed bit, second chance; Linux active/inactive, MGLRU |
| Belady | FIFO can fault more with more frames; LRU/OPT immune (stack algorithms) |
| Thrashing | Σ working sets > RAM; fix: working set / PFF / kill |
| mmap | file or anonymous; shared vs private (CoW); lazy; MAP_POPULATE |
| Pinning / mlock | "never evict this page" ⇒ no major faults; may still migrate; swap-off is not enough (code pages) |
| DMA pin | stays in RAM **and** fixed physical frame (device uses physical addresses) |
| HFT | mlockall + pre-touch (stack, pools) + huge pages + no malloc on hot path ⇒ no faults when trading |
| NVIDIA | pinned memory for DMA; unified memory faults → prefetch |
| Overcommit | malloc succeeds lazily; OOM killer when real frames run out |
| Example string 1 2 3 4 1 2 5 1 2 3 4 5 | 3 frames: FIFO 9, LRU 10, OPT 7; FIFO 4 frames: 10 |
