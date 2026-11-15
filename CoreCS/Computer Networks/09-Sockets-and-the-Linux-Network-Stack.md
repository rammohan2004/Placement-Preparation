# 09. Sockets & the Linux Network Stack

> **Asked at:** [HFT] what happens inside Linux when a packet arrives, where packets get dropped, RSS/IRQ affinity, busy polling, socket buffer sizing for market data, `send()` semantics · [NVIDIA] NIC rings, DMA, offloads (TSO/GRO/checksum), packet rates at 100 Gbps, XDP/AF_XDP · [QCOM] [TI] driver ↔ stack interface, NAPI, socket API in embedded Linux · [AMZN] [DESHAW] socket calls for a TCP server/client, listen backlog, `SO_REUSEPORT`, server architectures.
> **Prerequisites:** [06](06-Transport-Layer-UDP-and-TCP-Connections.md) (ports, handshake, accept queue), [07](07-TCP-Reliability-Flow-and-Congestion-Control.md) (buffers, Nagle). OS side: [OS 11](../Operating%20Systems/11-IO-Systems-IO-Models-and-Disk-Scheduling.md) (blocking / non-blocking / epoll / io_uring, top/bottom halves, NAPI, kernel bypass), [OS 13](../Operating%20Systems/13-Linux-Internals-and-Low-Latency-Tuning.md) (IRQ path, jitter, core isolation), [COA 13](../Computer%20Organization%20and%20Architecture/13-IO-Interrupts-DMA.md) (DMA, MSI-X).

---

## 1. Intuition
A **socket** is a mailbox that the kernel manages for your program. You don't touch the network card. You put letters in the mailbox (`send`) and take letters out (`recv`), and the kernel's postal workers do the rest:
- the network card drops incoming parcels into a **conveyor belt in RAM** (the RX ring);
- it rings a bell (an **interrupt**);
- a kernel worker unpacks the envelopes layer by layer (Ethernet → IP → TCP);
- the worker puts the contents into the right mailbox and wakes up whoever was waiting there.

Every step costs time, which is why HFT firms tune, or completely skip, this path.

---

## 2. Deep Dive

### 2.1 What a socket is
- From your program's view, a socket is a **file descriptor** (a small integer, like an open file; [OS 10](../Operating%20Systems/10-File-Systems.md)).
- In the kernel it is a `struct socket` + `struct sock`. These hold:
  - the protocol state (e.g. TCP state, cwnd, sequence numbers);
  - a **receive queue** (arrived data not yet read by the app);
  - a **send queue** (data written by the app but not yet sent/ACKed);
  - a **wait queue** of threads sleeping on the socket ([OS 04](../Operating%20Systems/04-CPU-Scheduling.md) wait queues).
- `socket(domain, type, protocol)`:

| Domain | Meaning |
|---|---|
| `AF_INET` / `AF_INET6` | IPv4 / IPv6 networking |
| `AF_UNIX` | **local** IPC between processes on the same machine (faster than TCP loopback: no TCP/IP processing) |
| `AF_PACKET` | raw Ethernet frames (tcpdump) |

| Type | Protocol |
|---|---|
| `SOCK_STREAM` | TCP (byte stream) |
| `SOCK_DGRAM` | UDP (datagrams) |
| `SOCK_RAW` | build your own IP packets (ping, traceroute; needs privileges) |

### 2.2 The socket API: TCP server and client
```
        SERVER                                   CLIENT
  socket()                                 socket()
  setsockopt(SO_REUSEADDR)                 (optional bind())
  bind(:8080)        ← claim the port
  listen(backlog)    ← state LISTEN, create SYN + accept queues (note 06 §2.5)
  accept()  ── blocks ──┐                  connect(server:8080)  ── kernel sends SYN, waits 1 RTT
                        │◄── handshake completes in the KERNEL ──►│
  returns NEW fd ───────┘                  returns 0
  recv()/send()  ◄──────── data ────────►  send()/recv()
  close()            ← FIN                 close()
```
```c
// Minimal TCP echo server (error handling trimmed for clarity)
int s = socket(AF_INET, SOCK_STREAM, 0);
int one = 1;
setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &one, sizeof one);   // restart without "address in use"
struct sockaddr_in a = { .sin_family = AF_INET,
                         .sin_port = htons(8080),              // network byte order!
                         .sin_addr.s_addr = htonl(INADDR_ANY) };// all interfaces (0.0.0.0)
bind(s, (struct sockaddr *)&a, sizeof a);
listen(s, 1024);                                              // accept-queue size
for (;;) {
    int c = accept(s, NULL, NULL);       // takes one ESTABLISHED connection from the accept queue
    char buf[4096]; ssize_t n;
    while ((n = recv(c, buf, sizeof buf, 0)) > 0)   // 0 ⇒ peer sent FIN
        send_all(c, buf, n);             // loop until all n bytes are queued
    close(c);
}
```
```c
// Client
int s = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in srv = { .sin_family = AF_INET, .sin_port = htons(8080) };
inet_pton(AF_INET, "10.0.0.5", &srv.sin_addr);
connect(s, (struct sockaddr *)&srv, sizeof srv);   // kernel picks an ephemeral port, does the handshake
setsockopt(s, IPPROTO_TCP, TCP_NODELAY, &one, sizeof one);   // latency-sensitive: no Nagle (note 07)
send(s, msg, len, 0);
```
**UDP:** `socket(AF_INET, SOCK_DGRAM, 0)`, then `bind()` to a port to receive, and `sendto()`/`recvfrom()` with an address per datagram. A **"connected" UDP socket** (`connect()` on UDP sends nothing on the wire) fixes the peer address: it allows plain `send()`/`recv()`, filters out other senders, and lets ICMP errors reach you.

**Semantics you must get right:**
- **Byte order:** network headers are **big-endian** ("network byte order"). x86 and ARM are little-endian. Always use `htons/htonl` (host → network) and `ntohs/ntohl` for ports and addresses.
- **`send()` returning success means the data was copied into the kernel's send buffer, not delivered.** It may also accept **fewer bytes** than asked (non-blocking sockets, signals), so loop (`send_all`).
- **`recv()` may return fewer bytes** than a whole message (TCP byte stream, note 06 §2.4). A return of 0 means the peer closed.
- A blocking `send()` sleeps when the send buffer is full (the receiver or network is slow). This is TCP back-pressure reaching your thread.

### 2.3 Serving many connections (architectures)
| Model | How | Trade-off |
|---|---|---|
| Thread (or process) per connection | `accept()` → spawn a thread with blocking I/O | simple; 10 K threads ⇒ memory + context-switch cost |
| **Event loop** | non-blocking sockets + **epoll**; one thread handles thousands | scalable (nginx, Redis, Node.js); never block the loop |
| **N event loops, one per core** | each thread has its own epoll + its own listening socket via **`SO_REUSEPORT`** (the kernel load-balances new connections across them) | scales across cores with no shared accept lock |
| **io_uring** | submit and complete I/O through shared rings; fewer syscalls | highest throughput on modern Linux |
| Kernel bypass | the app polls the NIC directly (DPDK/ef_vi/OpenOnload) | lowest latency; lose the kernel's tools/protections ([OS 11 §2.9](../Operating%20Systems/11-IO-Systems-IO-Models-and-Disk-Scheduling.md)) |
Details of blocking / non-blocking / epoll (level vs edge triggered) / io_uring are in [OS 11 §2.5–2.7](../Operating%20Systems/11-IO-Systems-IO-Models-and-Disk-Scheduling.md).

### 2.4 The receive path: what happens when a packet arrives
First the terms:
- **RX ring (receive ring):** a circular array of **descriptors** in RAM, shared between the driver and the NIC. Each descriptor points to an empty buffer the driver pre-allocated. The NIC writes a packet into the next buffer by **DMA** (direct memory access, without the CPU) and marks the descriptor "done".
- **MSI-X interrupt:** a per-queue interrupt the NIC sends to a specific CPU.
- **NAPI:** Linux's "interrupt once, then poll" scheme. After the first interrupt, the driver **disables that queue's interrupt** and polls the ring in a batch, so that heavy traffic doesn't cause an interrupt per packet ([OS 11 §2.2](../Operating%20Systems/11-IO-Systems-IO-Models-and-Disk-Scheduling.md)).
- **softirq:** the kernel's deferred-work mechanism (a "bottom half"). It runs soon after the interrupt, with interrupts enabled. Network receive processing runs in the `NET_RX` softirq.
- **sk_buff (skb):** the kernel's metadata structure for one packet: pointers to its headers and data, device, timestamps, and so on. It's passed from layer to layer **without copying the data**.

```
 wire ─► NIC: check CRC, MAC filter, (RSS hash ⇒ choose RX queue)
         │ DMA packet into the next free buffer of that queue's RX ring
         │ raise MSI-X interrupt to the CPU assigned to that queue
         ▼
 CPU: hard IRQ handler (tiny): ack, schedule NAPI, disable queue IRQ            ~1 µs
         ▼
 NET_RX softirq: NAPI poll — up to "budget" (64) packets from the ring
         │ for each: build an skb; GRO (merge consecutive TCP segments)
         ▼
 IP layer: validate header; netfilter hooks (iptables/nftables firewall, conntrack, NAT);
         │ routing decision: for this host? (else forward)
         ▼
 TCP/UDP: look up the socket by 5-tuple (hash table); checksum (often already done by NIC);
         │ TCP: sequence checks, reorder buffer, generate ACK, update windows
         ▼
 append the data to the socket's RECEIVE QUEUE; wake threads on its wait queue
         ▼ (scheduler runs the woken thread: wake-up + context switch, a few µs)
 app: recv() COPIES data from kernel buffer → user buffer; returns
```
When the ring is empty, NAPI re-enables the interrupt. If the softirq has too much work, it continues in the per-CPU `ksoftirqd` kernel thread, which competes with apps for that CPU.

### 2.5 The transmit path
```
 app: send()  → COPY user data into skbs in the socket SEND QUEUE; returns (data not yet on the wire!)
 TCP: may send now if cwnd/rwnd allow (else it waits for ACKs); segments, headers, (TSO: one big skb)
 IP:  routing lookup, netfilter, fragmentation check
 neighbour layer: MAC of the next hop (ARP cache, note 02)
 qdisc (queueing discipline): per-device queue + scheduling (fq = per-flow fair queuing + pacing)
 driver: put a descriptor in the TX RING; ring the NIC's "doorbell" register (MMIO write)
 NIC: DMA-reads the packet from RAM, computes checksums/segments (offloads), transmits
 NIC: TX-completion interrupt ⇒ driver frees the buffer
 TCP: keeps the data in the send queue until it's ACKed (needed for retransmission)
```

### 2.6 Using many cores: multi-queue NICs, RSS and affinity
A single core can't process 10–100 Gbps, so modern NICs have **many RX/TX queues**:
- **RSS** (Receive Side Scaling): the NIC **hashes the 5-tuple** and picks an RX queue from the hash. Each queue has its own interrupt, mapped to a CPU. All packets of one flow land on **the same queue and core**, which avoids reordering and keeps that flow's data in one core's cache.
- **RPS / RFS** (Receive Packet/Flow Steering): software versions of RSS. RFS steers a flow's processing to the core **where the consuming application thread runs**, for cache locality. **XPS** picks TX queues per CPU.
- **IRQ affinity:** `/proc/irq/<n>/smp_affinity`. `irqbalance` sets it automatically. Latency-critical setups set it **by hand**.
- **[HFT] setup:** pin the market-data queue's IRQ, and the thread that reads it, to cores on the **same NUMA node as the NIC** (PCIe slot locality). Isolate those cores from other work, and **busy-poll** (`SO_BUSY_POLL` / `net.core.busy_poll`) so the app spins on the queue instead of sleeping and waiting for an interrupt ([OS 13 §2.6](../Operating%20Systems/13-Linux-Internals-and-Low-Latency-Tuning.md)).

### 2.7 NIC offloads: moving per-packet work into hardware
| Offload | What the NIC does | Effect |
|---|---|---|
| **Checksum offload** | computes/verifies IP/TCP/UDP checksums | saves CPU per byte |
| **TSO / GSO** (TCP/Generic Segmentation Offload) | the kernel hands the NIC **one large (≤ 64 KB) "super-segment"**; the NIC cuts it into MSS-sized packets | the stack processes 1 packet instead of ~45 |
| **GRO / LRO** (Generic/Large Receive Offload) | **merges** consecutive received segments of one flow into a big skb before the stack sees it | fewer per-packet traversals up the stack |
| **Interrupt coalescing** (`ethtool -C`) | wait for N packets or T µs before interrupting | throughput ↑, **latency ↑**: HFT sets it to the minimum or off |
| **Hardware timestamping** (`SO_TIMESTAMPING`) | stamps packets at the NIC with a PTP-synchronised clock | exact wire-time measurement [HFT] (note 10) |
| VLAN tag insert/strip, RSS hashing, (TLS/crypto offload on newer NICs) | | |
- **Trap:** `tcpdump` on the host shows **"packets" of 20–64 KB**, larger than the MTU. That's TSO/GRO: tcpdump sees the super-segments before the NIC splits them (on TX) or after GRO merged them (on RX). Not a bug.
- **[HFT]** often **disables GRO/LRO and coalescing** on market-data interfaces: merging and waiting add latency, and every message must be seen individually and immediately.

### 2.8 Buffers, and where packets get dropped (and how to see it)
| Where | Why it drops | How to see it | Fix |
|---|---|---|---|
| NIC RX ring | softirq/CPU didn't drain the ring fast enough (burst, busy core) | `ethtool -S eth0` (`rx_missed`, `rx_no_buffer`, `rx_fifo_errors`) | bigger ring (`ethtool -G eth0 rx 4096`), more queues/cores, IRQ affinity, busy poll |
| Per-CPU backlog (RPS) | softirq queue full | `/proc/net/softnet_stat` (2nd column = dropped; 3rd = "time squeeze": the budget ran out) | `netdev_max_backlog`, `netdev_budget` |
| Firewall / conntrack | rules, full connection-tracking table | `nstat`, `dmesg` ("nf_conntrack: table full") | rules, `nf_conntrack_max`, skip conntrack for the hot path |
| **Socket receive buffer** | the **application didn't read fast enough** | `netstat -su` / `nstat` (**UDP `RcvbufErrors`**), `ss -m` (memory per socket) | larger `SO_RCVBUF` (capped by `net.core.rmem_max`), read faster / in a dedicated thread |
| TCP | the receive buffer full ⇒ rwnd = 0 (no drop: back-pressure) | `ss -tni`, Wireshark "zero window" | read faster, bigger buffer |
- **UDP has no back-pressure:** if the app is slow, the socket buffer fills and **new datagrams are silently dropped**. For **market data**, size `SO_RCVBUF` to absorb the worst-case burst during the worst-case app stall (N3) [HFT].
- Linux **doubles** the `SO_RCVBUF` value you set (to account for bookkeeping) and caps it at `rmem_max`. TCP **auto-tunes** its buffers within `tcp_rmem`; setting `SO_RCVBUF` manually disables that auto-tuning.

### 2.9 Socket options worth knowing
| Option | Purpose |
|---|---|
| `SO_REUSEADDR` | bind a port that still has TIME_WAIT connections (server restart) |
| `SO_REUSEPORT` | several sockets bind the **same** port; the kernel spreads connections/datagrams across them (one per core) |
| `TCP_NODELAY` | disable Nagle (note 07) |
| `TCP_QUICKACK` | ACK immediately (Linux; re-arm after reads) |
| `TCP_CORK` | hold partial segments until uncorked (bulk senders) |
| `SO_KEEPALIVE` + `TCP_KEEPIDLE/INTVL/CNT` | detect dead peers (note 06) |
| `TCP_USER_TIMEOUT` | fail a connection if data stays unACKed this long (faster dead-peer detection) |
| `SO_RCVBUF` / `SO_SNDBUF` | socket buffer sizes |
| `SO_LINGER` | `close()` behaviour; linger 0 ⇒ send RST and discard |
| `SO_BUSY_POLL` | spin on the device queue for N µs on `recv`/`epoll` instead of sleeping [HFT] |
| `SO_TIMESTAMPING` | software/hardware RX/TX timestamps [HFT] |
| `IP_ADD_MEMBERSHIP`, `IP_MULTICAST_IF/TTL/LOOP` | join multicast groups, choose the interface (note 10) |
| `SO_RCVLOWAT` | wake the reader only when ≥ N bytes are available |

### 2.10 eBPF and XDP (name-level, increasingly asked)
- **eBPF:** small, verified programs loaded into the kernel at hook points, for tracing (`bpftrace`), filtering and networking (**Cilium** for Kubernetes).
- **XDP** (eXpress Data Path): an eBPF program that runs **in the driver, before an skb is even allocated**. It can drop, redirect or rewrite packets at tens of millions of packets per second per core. Uses: DDoS filtering, L4 load balancers (Facebook's **Katran**).
- **AF_XDP:** an XDP program redirects chosen packets straight to a user-space ring, giving near-DPDK speed while the NIC stays under kernel control. It's a middle ground between the full stack and full kernel bypass.

### 2.11 Tools (symptom → command)
| Want | Command |
|---|---|
| connections, states, cwnd/rtt/retransmits per socket | `ss -tanpi`, `ss -s` |
| interfaces, addresses, routes, neighbours | `ip addr`, `ip route`, `ip neigh` |
| NIC stats, rings, offloads, coalescing, queues | `ethtool -S / -g / -k / -c / -l eth0` |
| protocol counters (retransmits, UDP buffer errors) | `nstat -az`, `netstat -s` |
| packets on the wire | `tcpdump -i eth0 -nn port 443 -w cap.pcap` + Wireshark |
| throughput test | `iperf3 -c host` |
| where CPU time goes in the stack | `perf top`, `perf record -g`, `bpftrace` |
| softirq load per CPU | `mpstat -P ALL`, `/proc/softirqs`, `/proc/net/softnet_stat` |

### 2.12 How to read this note: must know / name-only / skip
| Must know | Name-only | Skip |
|---|---|---|
| socket = fd + kernel buffers; server/client call sequence and what each does in the kernel; byte order; send/recv semantics; architectures (thread vs epoll vs SO_REUSEPORT); RX path step by step (ring, DMA, IRQ, NAPI, softirq, skb, socket lookup, wake-up, copy); TX path; RSS + affinity; TSO/GRO/checksum offload + coalescing trade-off; drop points + how to see them; UDP has no back-pressure; key socket options | RPS/RFS/XPS details, qdisc types, eBPF/XDP/AF_XDP, conntrack | driver code, skb field layout, netfilter internals |

---

## 3. Worked Numericals

**N1. Packet rates at line rate.** 10 Gbps Ethernet.
- Minimum frames: 64 B + 8 preamble + 12 gap = 84 B on the wire ⇒ 10¹⁰ / (84 × 8) = **14.88 million packets/s** ⇒ a budget of **67 ns per packet**: about 200 CPU cycles at 3 GHz, less than one cache miss to DRAM.
- Full-size frames (1538 B on the wire) ⇒ 10¹⁰ / (1538 × 8) ≈ **813 K packets/s** ⇒ 1.23 µs per packet.
- At 100 Gbps, ×10. The kernel stack costs on the order of ~1 µs of CPU per packet per core, which is why multi-queue + RSS, offloads, XDP or kernel bypass are needed.

**N2. RX ring overflow during a burst.** Market data bursts at 1 M packets/s. The core handling the queue is stalled for **1 ms** (a higher-priority softirq, or an SMI). The ring has 512 descriptors.
- Packets arriving during the stall = 10⁶ × 10⁻³ = **1000** > 512 ⇒ **~488 dropped** at the NIC (`rx_missed`).
- Ring of 4096 ⇒ absorbs it. Better still: an isolated core, so the stall doesn't happen.

**N3. UDP socket buffer sizing [HFT].** Feed peak 150 MB/s; the application can stall for up to **8 ms** (a page fault, logging, a GC pause in a JVM component).
- Bytes arriving during the stall = 150 × 10⁶ × 0.008 = **1.2 MB**. Linux's default receive buffer (~208 KB) ⇒ drops (`RcvbufErrors`).
- Set `SO_RCVBUF` ≥ ~2 MB (raise `net.core.rmem_max` first; the kernel stores double). And remove the stall itself (pre-fault memory, async logging; [OS 09](../Operating%20Systems/09-Virtual-Memory-Page-Faults-and-Replacement.md) §2.7).

**N4. Copy cost.** Receiving a steady 10 Gbps = 1.25 GB/s of data through `recv()`. Memory copy throughput ~10 GB/s per core.
- The copy alone ≈ 1.25 / 10 = **12.5 % of a core**, before any per-packet stack processing. That's why zero-copy designs exist (`MSG_ZEROCOPY`, io_uring, kernel bypass, RDMA in note 11).

**N5. Effect of TSO.** An app writes 64 KB; MSS = 1448 B (1460 − 12 B of timestamp option).
- Segments = ⌈65,536 / 1448⌉ = **46** packets. Without TSO, the stack builds and traverses 46 packets; with TSO, **1** large skb that the NIC slices. Per-packet stack work drops ~46×.

**N6. Kernel vs bypass latency budget (illustrative, one-way receive, small UDP packet).**
| Step | Kernel, interrupt-driven | Kernel + busy poll | Kernel bypass (ef_vi/DPDK) |
|---|---|---|---|
| NIC → RAM (DMA) | ~0.5 µs | ~0.5 µs | ~0.5 µs |
| Interrupt + softirq + IP/UDP | 1–2 µs | ~1 µs (no IRQ) | — (app polls the ring) |
| Wake-up + context switch | 2–5 µs | — (thread already spinning) | — |
| Copy to user | ~0.1 µs | ~0.1 µs | — (reads the DMA buffer) |
| **Total** | **~4–8 µs** (+ jitter) | **~2 µs** | **< 1 µs** |
Numbers vary by NIC, CPU and kernel; the *ordering* is the point to remember.

**Try it:** a 25 Gbps link receives 200-byte frames (+20 B preamble/gap) at line rate, and RSS spreads them evenly over 8 queues/cores. (a) Packets per second per core? (b) The time budget per packet per core?
<details><summary>Answer</summary>Total = 25 × 10⁹ / (220 × 8) ≈ 14.2 M packets/s ⇒ (a) ≈ <b>1.78 M packets/s per core</b>; (b) ≈ <b>562 ns per packet</b>.</details>

---

## 4. Interview Questions

### Basic
- **Q: Which system calls does a TCP server make? A client?** Server: socket → setsockopt(SO_REUSEADDR) → bind → listen → accept (loop) → recv/send → close. Client: socket → connect → send/recv → close (§2.2).
- **Q: What does `listen(fd, backlog)` do? What does `accept()` do?** listen: moves the socket to LISTEN and sizes the accept queue. accept: returns a new fd for an already-established connection (the handshake was done by the kernel).
- **Q: Why `htons()`?** Network headers are big-endian; host CPUs are usually little-endian.

### Intermediate
- **Q: `send()` returned 100. Did the peer receive 100 bytes?** No. 100 bytes were copied into the kernel send buffer. Delivery is asynchronous; an application-level ACK is the only proof of receipt.
- **Q: How does a server handle 10 K+ connections?** Non-blocking sockets + epoll event loops, one per core via SO_REUSEPORT (or io_uring), instead of a thread per connection (§2.3).
- **Q: What is RSS and why does it hash the 5-tuple?** It spreads flows across RX queues/cores; hashing per flow keeps each flow on one core ⇒ no reordering, warm caches.
- **Q: What are TSO and GRO? Why does tcpdump show 64 KB packets?** Segmentation/merging done by the NIC/driver; tcpdump sees the large pre-split or post-merged buffers.
- **Q: AF_UNIX vs TCP over 127.0.0.1?** AF_UNIX skips TCP/IP processing (no checksums, segmentation, ACKs) ⇒ lower latency and higher throughput; it can also pass fds and credentials.

### Advanced / killer follow-ups
- **Q [HFT] [NVIDIA]: Walk me through what happens from a packet hitting the NIC to `recv()` returning.** §2.4: RSS queue → DMA into the RX ring → MSI-X → hard IRQ → NAPI softirq poll → skb + GRO → IP + netfilter + routing → socket lookup by 5-tuple → TCP/UDP processing → receive queue → wake-up → `recv` copy. Then name the costs: IRQ, softirq, wake-up, copy.
- **Q [HFT]: Your market-data process is missing UDP packets. How do you find where?** Check, in order: NIC (`ethtool -S` missed/no_buffer ⇒ ring or CPU), `softnet_stat` (backlog), conntrack/firewall, socket (`nstat` UdpRcvbufErrors ⇒ app too slow / buffer too small), the app's own sequence-gap counters. Fix the layer that drops (§2.8, N2, N3).
- **Q [HFT]: How do you cut kernel receive latency without full kernel bypass?** Pin IRQ and thread to the same isolated, NUMA-local core; busy poll (SO_BUSY_POLL); interrupt coalescing off; GRO off; avoid conntrack; big rings; `TCP_NODELAY`/`QUICKACK`; or AF_XDP / OpenOnload.
- **Q [NVIDIA]: Why can't a single core handle 100 Gbps of small packets through the kernel?** ~148 Mpps at minimum size ⇒ ~7 ns per packet, while the stack needs ~µs. Hence multi-queue + RSS across cores, offloads, XDP, DPDK, or RDMA (which moves the transport into the NIC).
- **Q: What is NAPI and why does it exist?** Interrupt on the first packet, then poll the ring in batches with interrupts off; avoids interrupt storms/livelock at high rates; re-enables interrupts when idle.
- **Q: Where does XDP sit relative to iptables and DPDK?** XDP runs in the driver before skb allocation (much earlier and cheaper than netfilter), while the kernel keeps the NIC. DPDK takes the whole NIC into user space. AF_XDP sits in between.
- **Q: When would you set SO_RCVBUF manually, and what's the catch?** UDP feeds that must absorb bursts (N3). The catch: the value is capped by rmem_max and doubled internally, and for TCP it disables auto-tuning.

---

## 5. Common Traps
- Believing `send()` success = delivered.
- Forgetting partial `send()`/`recv()` (looping is required).
- Forgetting `htons` on the port: it binds to the wrong port number silently.
- Not setting `SO_REUSEADDR` on servers.
- Blaming "the network" for UDP loss that is really the socket buffer or the NIC ring on your own host.
- Thinking interrupt coalescing or GRO are always good (they trade latency for throughput).
- Assuming one core can process line rate: check the packets-per-second budget (N1).

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Socket | fd → kernel struct with protocol state, RX/TX queues, wait queue |
| Server calls | socket, setsockopt(REUSEADDR), bind, listen(backlog), accept → new fd |
| Client calls | socket, connect (kernel handshake), send/recv, close |
| Semantics | htons/ntohs; send = copied to kernel; partial send/recv; recv 0 = FIN |
| Architectures | thread per connection · epoll event loop · per-core loops + SO_REUSEPORT · io_uring · bypass |
| RX path | NIC (RSS) → DMA RX ring → MSI-X → hard IRQ → NAPI softirq → skb/GRO → IP/netfilter → socket lookup → TCP/UDP → receive queue → wake → copy |
| TX path | copy to send queue → TCP (cwnd/rwnd) → IP → ARP → qdisc → TX ring + doorbell → NIC DMA → completion |
| Scaling | multi-queue + RSS (5-tuple hash), IRQ affinity, RFS, NUMA locality |
| Offloads | checksum, TSO/GSO, GRO/LRO, coalescing (latency ↑), HW timestamps |
| Drops | ring (`ethtool -S`) · backlog (`softnet_stat`) · socket (`RcvbufErrors`) |
| UDP | no back-pressure ⇒ size SO_RCVBUF for burst × stall |
| HFT | pin + isolate + NUMA-local, busy poll, coalescing/GRO off, NODELAY, bypass/AF_XDP |
| Rates | 10 G min frames = 14.88 Mpps = 67 ns per packet |
