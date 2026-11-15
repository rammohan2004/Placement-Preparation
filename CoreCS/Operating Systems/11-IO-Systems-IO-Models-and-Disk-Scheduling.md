# 11. I/O Systems, I/O Models & Disk Scheduling

> **Asked at:** [HFT] blocking vs non-blocking vs epoll vs busy polling, kernel bypass, zero-copy · [AMZN] [DESHAW] select/poll/epoll, C10K, disk scheduling numericals, RAID levels · [NVIDIA] [QCOM] device drivers, interrupt top/bottom halves, DMA, ioctl/mmap · [TI] polling vs interrupts in firmware.
> **Prerequisites:** [01 OS Basics](01-OS-Basics-Kernel-and-System-Calls.md), [COA 13 I/O, Interrupts & DMA](../Computer%20Organization%20and%20Architecture/13-IO-Interrupts-DMA.md) (MMIO, interrupt hardware, DMA engines).

---

## 1. Intuition
Waiting for I/O is like waiting for a pizza delivery:
- **Blocking:** stand at the door until it arrives (you do nothing else).
- **Non-blocking:** open the door every minute to check (busy, but you can do other things in between).
- **Multiplexing (epoll):** one doorbell for 10,000 deliveries; sleep until *any* bell rings, then see which.
- **Async (io_uring):** "leave it on the table and text me". You're told when it's done, with the pizza already inside.
- **Kernel bypass (HFT):** build your own door to the street so you don't go through the building lobby (kernel) at all.

---

## 2. Deep Dive

### 2.1 The kernel I/O stack
```
 Application           read(fd, buf, n) / send() / ioctl()
 ─────────────────────────────── syscall ───────────────────────────────
 VFS / socket layer    generic file & socket operations
 File system / TCP-IP  ext4, XFS … / protocol processing
 Block layer / netdev  I/O scheduler, request queues  / qdisc, NAPI
 Device driver         talks to the hardware: registers (MMIO), DMA descriptors, interrupts
 ─────────────────────────────── hardware ──────────────────────────────
 Device (NVMe SSD, NIC, GPU) ── DMA into RAM ──► raises an interrupt when done
```
**Device classes:** **character** (byte stream: serial, keyboard, `/dev/null`), **block** (fixed-size blocks, random access, cached: disks), **network** (sockets, not files in `/dev`).

### 2.2 Polling vs interrupts vs DMA (OS view)
| | Polling (busy-wait) | Interrupt-driven | DMA |
|---|---|---|---|
| CPU work | loops reading a status register | does other work; ISR runs when the device signals | the device copies data itself; the CPU sets up descriptors and gets one interrupt at the end |
| Latency | **lowest** (no interrupt entry ~1–5 µs) | interrupt overhead | best for bulk data |
| CPU cost | 100 % of a core while waiting | low when events are rare, **high at very high event rates** (interrupt storm) | lowest per byte |
| Used by | HFT (busy-poll NICs), very fast NVMe, MCU tight loops | most devices | disks, NICs, GPUs |
**Hybrid:** Linux **NAPI** for NICs: an interrupt on the first packet, then switch to **polling** while packets keep coming (avoids interrupt storms), then back to interrupts when idle. **Interrupt coalescing:** the NIC raises one interrupt per N packets or per T µs (throughput ↑, latency ↑; HFT disables it).

### 2.3 Device drivers [NVIDIA][QCOM][TI]
A driver is kernel code (usually a loadable module) that:
1. Registers with a subsystem (PCI, USB, platform/device tree) and is **probed** when a matching device is found.
2. Maps device registers (`ioremap` → MMIO), sets up DMA buffers (`dma_alloc_coherent`, scatter-gather), and requests an IRQ (`request_irq`).
3. Exposes an interface: a char device with `file_operations` (`open`, `read`, `write`, **`ioctl`** for device-specific commands, **`mmap`** to map device memory or buffers into user space), or plugs into the block/network/DRM subsystem.

**Interrupt handling is split into two halves:**
| | Top half (hard IRQ handler) | Bottom half (deferred work) |
|---|---|---|
| Runs | immediately, interrupts (often) disabled on that line | later, with interrupts enabled |
| Must | be **very short**: acknowledge the device, grab data/status, schedule the bottom half | do the heavy processing |
| Can sleep? | ❌ | **softirq/tasklet: ❌** · **workqueue / threaded IRQ: ✅** (runs in a kernel thread) |
Example: NIC interrupt → top half acks it and schedules NAPI → softirq (`NET_RX`) processes packets → wakes the socket's reader.

**Talking to a device from user space:**
- `ioctl()` commands (e.g. the NVIDIA driver's control interface);
- `mmap()` of device memory (GPU BARs, framebuffers) ⇒ user space writes registers or buffers directly;
- **UIO / VFIO:** user-space drivers (DPDK uses VFIO to drive NICs from user space with the IOMMU for safety).

### 2.4 Buffering, caching, spooling
- **Buffering:** smooth speed mismatches (network → disk), handle different transfer sizes, and give **copy semantics** (the app can reuse its buffer after `write()` returns). **Double buffering:** fill one buffer while the other is processed.
- **Caching:** keep copies of data (page cache). Buffer = maybe the only copy; cache = a copy of data stored elsewhere.
- **Spooling:** queue output for a device that can serve one job at a time (printer).

### 2.5 The five I/O models (Stevens) [HFT]
| Model | What the thread does | Data copy | Typical API |
|---|---|---|---|
| **Blocking** | sleeps in `read()` until data arrives | kernel → user inside `read` | default sockets/files |
| **Non-blocking** | `read()` returns `EAGAIN` immediately if there's no data; the app retries (polls) | in `read` | `O_NONBLOCK` |
| **I/O multiplexing** | blocks in `select/poll/epoll_wait` until **any** of many fds is ready, then reads | in `read` | `select`, `poll`, **`epoll`** (Linux), `kqueue` (BSD/macOS) |
| Signal-driven | kernel sends SIGIO when ready | in `read` | rarely used |
| **Asynchronous** | submit the request; get notified on **completion** (data already in your buffer) | done by the kernel before notification | **`io_uring`**, POSIX AIO, Windows IOCP |
**Readiness vs completion:** epoll says "you can read now without blocking" (you still call `read`). io_uring says "the read is done, the data is in your buffer".

### 2.6 select vs poll vs epoll
| | `select` | `poll` | `epoll` |
|---|---|---|---|
| Interface | fd bitmaps (`FD_SETSIZE` = 1024 limit) | array of `pollfd` | kernel-side interest list: `epoll_create`, `epoll_ctl` (add/remove once), `epoll_wait` |
| Per-call cost | **O(n)**: copy all fds in, kernel scans all, app scans all | **O(n)** | **O(ready)**: only ready fds are returned |
| Scales to 100k connections? | ❌ | ❌ | ✅ (the C10K/C10M answer) |
```c
int ep = epoll_create1(0);
struct epoll_event ev = { .events = EPOLLIN, .data.fd = listen_fd };
epoll_ctl(ep, EPOLL_CTL_ADD, listen_fd, &ev);
struct epoll_event ready[64];
for (;;) {
    int n = epoll_wait(ep, ready, 64, -1);          // sleep until something is ready
    for (int i = 0; i < n; i++) handle(ready[i].data.fd);  // only ready fds
}
```
- **Level-triggered (default):** `epoll_wait` keeps reporting the fd while data remains ⇒ simpler.
- **Edge-triggered (`EPOLLET`):** reported only when new data **arrives** ⇒ you must read until `EAGAIN` (non-blocking fds), or you'll never be told again ⇒ fewer wake-ups, more care needed.
- This is the core of the **event loop** (nginx, Redis, Node.js, Netty): one thread, many connections, non-blocking fds + epoll.

### 2.7 io_uring (Linux 5.1+)
Two ring buffers shared between the app and the kernel: a **submission queue** (SQ) and a **completion queue** (CQ). The app writes requests into SQ entries and reads results from CQ entries, so many I/Os cost **one** syscall (`io_uring_enter`), or **zero** syscalls in SQPOLL mode (a kernel thread polls the SQ). It works for files, sockets, `accept`, and more. It's the modern answer for high-throughput async I/O.

### 2.8 Zero-copy
Sending a file over a socket with `read` + `write`:
```
disk ──DMA──► page cache ──CPU copy──► user buffer ──CPU copy──► socket buffer ──DMA──► NIC
   (1)                         (2)                        (3)                  (4)
4 copies (2 by the CPU), 4 mode switches (2 syscalls × enter/exit)
```
- **`sendfile(out_sock, in_file, …)`:** page cache → socket inside the kernel ⇒ no user copy; with NIC scatter-gather, **no CPU copy at all** (only the DMAs). Used by web servers and Kafka.
- `splice`/`vmsplice` (pipes), **`mmap` + `write`** (one CPU copy saved), `MSG_ZEROCOPY` for sends, `copy_file_range` (file → file, can be offloaded).

### 2.9 Kernel bypass [HFT]
The kernel network stack costs ~5–20 µs per packet round trip (interrupt, softirq, TCP/IP, socket, copy, wake-up). HFT moves the data path **out of the kernel**:
- **DPDK:** a user-space driver (via VFIO) **busy-polls** NIC rings mapped into the process. No interrupts, no syscalls, no copies.
- **Solarflare/AMD OpenOnload, Mellanox VMA / ef_vi:** a user-space TCP/UDP stack, socket API compatible (LD_PRELOAD).
- **RDMA:** the NIC writes directly into remote memory.
- Plus: pinned cores, huge pages, NUMA-local NIC, interrupt coalescing off. Wire-to-wire latency goes from ~10 µs to **~1–2 µs** (FPGA: hundreds of ns). See [COA 14](../Computer%20Organization%20and%20Architecture/14-Low-Latency-Hardware-for-HFT.md).

### 2.10 Disks: HDD access time and scheduling
**HDD access time** = seek time + rotational latency + transfer time.
- Rotational latency (average) = ½ rotation = 0.5 × 60 / RPM seconds (7200 RPM ⇒ 4.17 ms).
- Random 4 KB I/O ≈ ~10 ms ⇒ ~100 IOPS. Sequential ≈ 100–250 MB/s.

**Disk-scheduling algorithms** (minimise head movement):
| Algorithm | Rule | Notes |
|---|---|---|
| FCFS | in order of arrival | fair, lots of movement |
| **SSTF** | nearest request next | low movement; **starvation** of far requests |
| **SCAN** (elevator) | sweep to one **end of the disk**, reverse | uniform-ish wait |
| **C-SCAN** | sweep one way to the end, jump back to the start, sweep again | more uniform wait times |
| **LOOK / C-LOOK** | like SCAN / C-SCAN but turn at the **last request**, not the disk end | what's actually used in practice |
**SSDs:** no seek ⇒ these algorithms don't matter. Linux uses `none` or `mq-deadline` for NVMe (multi-queue block layer, many hardware queues). SSD details that come up: the FTL (flash translation layer), erase-before-write in large blocks, **write amplification**, TRIM, wear levelling. NVMe SSD: ~50–100 µs random read, hundreds of thousands of IOPS with deep queues.

### 2.11 RAID: many disks that look like one [AMZN][DESHAW]
**Problem:** one disk is **slow** (HDD ~200 MB/s, ~100–150 random IOPS) and **fails** (~1–2 % per year, taking its data with it).
**RAID (Redundant Array of Independent Disks):** put N disks behind a layer that shows the OS **one logical disk** (e.g. `/dev/md0`). The layer decides which physical disk each block goes to, which makes the logical disk faster, bigger, fault-tolerant, or a mix. The layer is either a **hardware RAID controller** or **software** (Linux `md`/`mdadm`, ZFS, Windows Storage Spaces). The file system above doesn't know.

**The three building blocks:**
- **Striping:** cut data into fixed chunks (e.g. 64 KB) and spread them round-robin across disks ⇒ one large I/O uses all disks in parallel.
  ```
  logical: A0 A1 A2 A3 A4 A5 …     Disk1: A0 A3   Disk2: A1 A4   Disk3: A2 A5    (~3× throughput)
  ```
- **Mirroring:** write identical copies to 2+ disks. Survives a disk loss; costs half the space.
- **Parity:** store the **XOR** of the data chunks in one extra chunk. Since x ⊕ x = 0, any **one** lost chunk = XOR of all the others.
  ```
  D0 = 1011, D1 = 0110, D2 = 1100  ⇒  P = D0 ⊕ D1 ⊕ D2 = 0001
  disk with D1 dies:  D1 = D0 ⊕ D2 ⊕ P = 1011 ⊕ 1100 ⊕ 0001 = 0110 ✓
  ```
  Protection for the price of **one** disk's space (vs half for mirroring).

**The levels = combinations of these:**
```
RAID 0 (stripe)     RAID 1 (mirror)    RAID 5 (stripe + rotating parity)    RAID 10 (mirror pairs, striped)
D1  D2  D3          D1  D2             D1  D2  D3  D4                       [D1 D2]   [D3 D4]
A0  A1  A2          A0  A0             A0  A1  A2  Pa                       A0 A0     A1 A1
A3  A4  A5          A1  A1             B0  B1  Pb  B2                       A2 A2     A3 A3
                                       C0  Pc  C1  C2     ← parity on a different disk each row
```
N disks of capacity C each:
| Level | Usable space | Survives | Reads | Small random write costs | Typical use |
|---|---|---|---|---|---|
| **0** striping | N·C | **0 failures**: one disk dies ⇒ all data lost | ~N× | 1 I/O | scratch, caches |
| **1** mirroring (usually 2 disks) | **C**: every disk holds the *same full copy*, so 2 disks ⇒ C and a 3-way mirror ⇒ still C | N − 1 (2 disks ⇒ 1) | up to N× | N I/Os (2 for a pair) | boot/OS disk |
| **5** stripe + 1 distributed parity (≥ 3 disks) | (N−1)·C | 1 | fast | **4 I/Os** | read-mostly storage |
| **6** stripe + 2 parities (≥ 4) | (N−2)·C | **any 2** | fast | **6 I/Os** | large-disk arrays |
| **10** mirror + stripe (≥ 4) | N·C / 2 | 1 per mirror pair | fast | 2 I/Os | **databases** |

- **RAID 1 vs RAID 10 space:** adding disks to RAID 1 adds **copies** (usable stays C, survives N − 1). Adding disks to RAID 10 adds **pairs** (usable = (N/2)·C, survives 1 per pair). So (N/2)·C is the RAID 10 formula, and both give C at N = 2.
- **Why "distributed" parity (RAID 5 vs RAID 4):** every write updates parity. One dedicated parity disk (RAID 4) would be hit by *every* write and become the bottleneck; rotating it spreads the load.
- **Small-write penalty (RAID 5):** to change one chunk, `new P = old P ⊕ old D ⊕ new D` ⇒ read old D, read old P, write new D, write new P = **4 I/Os per logical write** (RAID 6: 6). Hence databases prefer RAID 10.
- **Rebuild window:** after one failure, RAID 5 has **no** protection until the new disk is rebuilt by reading *every* other disk in full (4 TB ÷ ~150 MB/s ≈ 7.4 h). A second failure or unreadable sector then loses data ⇒ big arrays use RAID 6.
- **RAID is not a backup:** it protects against a disk dying, not against `rm`, bugs, corruption or ransomware (those reach every copy instantly).
- **Cloud [AMZN]:** generalised parity = **erasure coding** (Reed–Solomon, e.g. 10 data + 4 code chunks, any 4 can be lost) ⇒ object stores get high durability at ~1.4× space instead of 3× replication.
- SSD arrays use the same levels; only the "seek" part of the performance story disappears.

---

## 3. Worked Numericals

**Disk queue (classic):** cylinders 0–199, head at **53**, queue: 98, 183, 37, 122, 14, 124, 65, 67.

**N1. FCFS:** 53→98 (45) →183 (85) →37 (146) →122 (85) →14 (108) →124 (110) →65 (59) →67 (2) = **640 cylinders**.

**N2. SSTF:** 53→65 (12) →67 (2) →37 (30) →14 (23) →98 (84) →122 (24) →124 (2) →183 (59) = **236**.

**N3. SCAN (head moving toward 0):** 53→37 (16) →14 (23) →**0** (14) →65 (65) →67 (2) →98 (31) →122 (24) →124 (2) →183 (59) = **236** (= 53 down to 0 + 183 up).

**N4. C-SCAN (moving toward 199):** 53→65→67→98→122→124→183→**199** = 146; jump 199→0; 0→14→37 = 37.
Total = 146 + 37 = **183** if the return jump isn't counted, **382** if it is (199). State your convention.

**N5. LOOK / C-LOOK (moving up):**
- LOOK: 53→183 (130), reverse →14 (169) = **299**. (Moving down first: 53→14 (39) + 14→183 (169) = **208**.)
- C-LOOK: 53→183 (130), jump to 14, →37 (23) = **153** without the jump, **322** with it (169).

**N6. HDD access time.** Seek 5 ms, 7200 RPM, transfer 4 KB at 200 MB/s.
Rotational = 0.5 × 60/7200 s = 4.17 ms. Transfer = 4 KB / 200 MB/s ≈ 0.02 ms. Total ≈ **9.2 ms** ⇒ ≈ **109 random IOPS**. An NVMe SSD at ~80 µs ⇒ ~12,500 IOPS per queue at depth 1, ≈ 100× better.

**N7. Interrupts vs polling at high packet rates.** 1 M packets/s; interrupt cost 2 µs per packet ⇒ 2 s of CPU per second ⇒ **impossible on one core** (interrupt storm, livelock). NAPI/polling with batches of 64 ⇒ 15,625 interrupts/s ⇒ ~31 ms ⇒ 3 %. Busy polling: one core at 100 %, but the lowest latency.

**N8. RAID capacity, write IOPS and reliability.** 6 HDDs, each 4 TB, 150 random IOPS, failure rate 2 %/year.
| Level | Usable | Random reads/s | Random small writes/s |
|---|---|---|---|
| RAID 0 | 6 × 4 = **24 TB** | 6 × 150 = 900 | 900 / 1 = **900** |
| RAID 10 | 24 / 2 = **12 TB** | 900 | 900 / 2 = **450** |
| RAID 5 | 5 × 4 = **20 TB** | ≈ 900 | 900 / 4 = **225** |
| RAID 6 | 4 × 4 = **16 TB** | ≈ 900 | 900 / 6 = **150** |
(Writes: total disk I/Os per second ÷ I/Os each logical write costs.)
Reliability of RAID 0 with 4 disks: P(no disk fails in a year) = 0.98⁴ = 0.922 ⇒ **P(lose everything) ≈ 7.8 %/year**, about 4× a single disk. Striping alone *reduces* reliability.

**Try it:** head at 50, cylinders 0–199, queue 82, 170, 43, 140, 24, 16, 190. Total movement for SSTF and for SCAN moving toward 199.
<details><summary>Answer</summary>SSTF: 50→43 (7) →24 (19) →16 (8) →82 (66) →140 (58) →170 (30) →190 (20) = <b>208</b>. SCAN up: 50→82→140→170→190→199 = 149, then 199→43 (156) →24 (19) →16 (8) ⇒ 149 + 156 + 19 + 8 = <b>332</b> (= 149 + 183).</details>

---

## 4. Interview Questions

### Basic
- **Q: Polling vs interrupts?** §2.2. Polling has the lowest latency but burns a core; interrupts are efficient at low event rates.
- **Q: What is DMA and why is it needed?** The device moves data to/from RAM without the CPU copying each byte; the CPU gets one interrupt at the end.
- **Q: Blocking vs non-blocking I/O?** §2.5.

### Intermediate
- **Q: select vs poll vs epoll?** §2.6. O(n) scans and an fd limit vs O(ready) with a kernel interest list.
- **Q: Level- vs edge-triggered epoll?** §2.6. With ET you must drain the fd until `EAGAIN`.
- **Q: What is zero-copy? How does `sendfile` help?** §2.8.
- **Q: Top half vs bottom half?** §2.3 table.
- **Q: SSTF vs SCAN? Which starves?** SSTF can starve far requests; SCAN/LOOK bound the wait.

### Advanced / killer follow-ups
- **Q [HFT]: Why busy-poll instead of epoll in a trading system?** epoll means sleeping and being woken (interrupt → softirq → wake-up → context switch ≈ several µs, plus jitter). Busy-polling a NIC ring (kernel bypass) reacts within ~100s of ns at the cost of a dedicated core.
- **Q [HFT]: What is kernel bypass? Name technologies.** §2.9: DPDK, OpenOnload/ef_vi, VMA, RDMA. A user-space driver/stack with polling, no syscalls or copies.
- **Q: epoll vs io_uring?** Readiness vs completion; io_uring batches many operations per syscall (or none with SQPOLL) and supports regular files properly (epoll doesn't help with disk files: they're always "ready").
- **Q [NVIDIA][QCOM]: How does a user-space program talk to a GPU/accelerator?** Through the driver's device file: `ioctl` for control, `mmap` of command buffers/device memory, DMA for bulk data, interrupts or fences for completion.
- **Q: Why can't you do heavy work in an interrupt handler?** Interrupts may be disabled (other devices delayed), you can't sleep, and it adds latency to everything ⇒ defer to softirq/workqueue/threaded IRQ.
- **Q: What is RAID? Compare 0, 1, 5, 6 and 10.** Many disks shown as one logical disk, built from striping (speed), mirroring (copies) and parity (XOR rebuilds one lost chunk). See the §2.11 table: 0 = fast, no protection; 1 = mirror; 5 = (N−1) space, survives 1; 6 = survives 2; 10 = mirror + stripe, half space, best for writes. *Follow-up:* "How does RAID 5 rebuild a dead disk?" XOR the surviving chunks of each stripe (D_lost = XOR of the others, including parity).
- **Q: Why do databases prefer RAID 10 over RAID 5?** The RAID-5 small-write penalty: one logical write = read old data + read old parity + write both = 4 I/Os (RAID 10: 2). Also rebuilds are faster and less risky (copy one mirror instead of reading every disk).
- **Q: Is RAID a backup?** No. It survives disk failure only; deletes, corruption and ransomware hit all copies immediately.
- **Q [AMZN]: How do cloud object stores get durability cheaper than 3 copies?** Erasure coding (generalised parity, e.g. 10 + 4 Reed–Solomon chunks across machines/racks): survives 4 losses at 1.4× space.
- **Q: Do disk-scheduling algorithms matter for NVMe?** Barely: no seek time; deep parallel hardware queues; Linux uses `none`/`mq-deadline`.

---

## 5. Common Traps
- Saying epoll is "asynchronous I/O". It's readiness notification; the read is still synchronous.
- Using edge-triggered epoll with blocking fds, or without draining ⇒ hangs.
- Forgetting that disk files are always "readable" for select/epoll.
- Counting (or not counting) the C-SCAN return jump without stating it.
- Doing heavy processing in a top-half ISR.
- Assuming interrupts are always better than polling (not at very high rates or for lowest latency).
- Calling RAID 0 "redundant". It has no redundancy, and with N disks it fails ~N× more often than one disk.
- Forgetting the RAID-5 write penalty (4 I/Os per small write) when estimating write IOPS.
- Saying "RAID means we don't need backups."

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| I/O stack | app → syscall → VFS/socket → FS/TCP → block/netdev → driver → device (DMA + IRQ) |
| Polling / IRQ / DMA | lowest latency + burns CPU / efficient at low rates / bulk transfer |
| NAPI | IRQ then poll under load |
| Driver | probe, ioremap, DMA, request_irq; `file_operations`: read/write/ioctl/mmap |
| IRQ halves | top: ack + schedule (no sleep); bottom: softirq/tasklet (no sleep) or workqueue/threaded (can sleep) |
| I/O models | blocking, non-blocking, multiplexing, signal-driven, async |
| epoll | O(ready), interest list; LT vs ET (drain to EAGAIN) |
| io_uring | SQ/CQ rings, batch or zero syscalls, completion-based |
| Zero-copy | sendfile/splice: no user copy; read+write = 4 copies |
| Kernel bypass | DPDK, OpenOnload, RDMA; busy-poll; ~1–2 µs |
| HDD time | seek + ½ rotation + transfer ≈ 10 ms ⇒ ~100 IOPS |
| Disk scheduling | FCFS 640, SSTF 236, SCAN 236 (classic queue, head 53) |
| SSD | no seek; FTL, write amplification, TRIM; `none`/`mq-deadline` |
| RAID building blocks | striping (parallel speed) · mirroring (copies) · parity (XOR rebuilds 1 lost chunk) |
| RAID levels | 0: N·C, survives 0 · 1: C (N copies), survives N−1 · 5: (N−1)C, survives 1, write = 4 I/Os · 6: (N−2)C, survives 2, 6 I/Os · 10: N·C/2, DBs · not a backup · cloud = erasure coding |
