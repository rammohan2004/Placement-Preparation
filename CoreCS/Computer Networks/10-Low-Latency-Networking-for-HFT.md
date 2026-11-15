# 10. Low-Latency Networking for HFT

> **Asked at:** [HFT] (IMC, Optiver, Tower, Graviton, Quadeye) why market data is UDP multicast, IGMP, A/B line arbitration, gap detection and recovery, order entry over TCP, tick-to-trade budget, microwave vs fibre, PTP time sync and hardware timestamps, microbursts, cut-through switches, "design a feed handler" · [DESHAW] the same at concept level · [NVIDIA] multicast and time sync in clusters (awareness).
> **Prerequisites:** [01](01-Network-Basics-and-Layered-Models.md) (propagation vs serialization), [02](02-Data-Link-Layer-and-Ethernet.md) (multicast MAC, cut-through), [03](03-Reliable-Data-Transfer-and-Sliding-Window.md) §2.8 (why no inline retransmission), [06](06-Transport-Layer-UDP-and-TCP-Connections.md) (TCP vs UDP), [07](07-TCP-Reliability-Flow-and-Congestion-Control.md) (Nagle, slow start after idle), [09](09-Sockets-and-the-Linux-Network-Stack.md) (RX path, busy poll, drops). Hardware side: [COA 14](../Computer%20Organization%20and%20Architecture/14-Low-Latency-Hardware-for-HFT.md); OS side: [OS 11 §2.9](../Operating%20Systems/11-IO-Systems-IO-Models-and-Disk-Scheduling.md) (kernel bypass), [OS 13](../Operating%20Systems/13-Linux-Internals-and-Low-Latency-Tuning.md) (jitter).

---

## 1. Intuition
An exchange is an auctioneer shouting prices into a room through loudspeakers (**multicast market data**). Traders whisper orders to the auctioneer through private phone lines (**TCP order entry**). Whoever **hears a price change first and replies first** wins the trade. So trading firms:
- put their computers in the **same building** as the exchange (colocation);
- use the **straightest, fastest** cables between cities (microwave towers beat fibre);
- skip every unnecessary layer (kernel bypass, cut-through switches, FPGAs);
- measure everything in **nanoseconds** with precisely synchronised clocks (PTP).

---

## 2. Deep Dive

### 2.1 The anatomy of a trading network
```
          EXCHANGE (colocation data centre)
  ┌────────────────────────────────────────────┐
  │ Matching engine ──► market-data publisher ──┼─► UDP multicast, feeds A and B (two networks)
  │        ▲                                    │
  │ Order gateways ◄────────────────────────────┼── TCP sessions (FIX or binary protocol)
  └────────────────────────────────────────────┘
           │  (cross-connect cables, metres)
  ┌────────▼───────────────────────────────────────────────────────────┐
  │ FIRM's rack:  L1/cut-through switch → NIC (kernel bypass) → feed    │
  │ handler (build order book) → strategy → order gateway → NIC → switch│
  └─────────────────────────────────────────────────────────────────────┘
```
- **Matching engine:** the exchange's program that matches buy and sell orders.
- **Order book:** for one instrument, the list of resting buy orders (bids) and sell orders (asks) at each price.
- **Colocation ("colo"):** renting rack space inside the exchange's data centre, so the propagation delay is metres of cable (ns), not kilometres (ms). Exchanges often make every customer's cable the **same length**, for fairness.
- **Tick-to-trade latency:** from the market-data packet ("tick") hitting your NIC to your order leaving your NIC. It's the key metric: ~**1–5 µs** in software with kernel bypass, **tens to hundreds of ns** on FPGAs. **Wire-to-wire** = the same thing, measured at the network wire on both sides.

### 2.2 Why market data is UDP multicast
- **Multicast** = one sender transmits **one copy** to a **group address** (IPv4 224.0.0.0/4). The network (switches, routers) **replicates** the packet only where receivers have joined.
- Why not TCP unicast to each customer?
  1. The exchange would send N copies, one after another, so customer #1 gets the data before customer #500 (**unfair**).
  2. A slow receiver back-pressures TCP; one stuck customer must not delay others.
  3. Retransmission would stall later data behind a loss (head-of-line blocking, note 07).
  4. Multicast is efficient: one copy per link.
- **How a host joins a group:**
  1. The app calls `setsockopt(IP_ADD_MEMBERSHIP, group, interface)`.
  2. The kernel sends an **IGMP** (Internet Group Management Protocol) **Membership Report**: "this host wants group 233.54.12.1".
  3. Switches doing **IGMP snooping** note which ports want which group and forward the group only there. Routers use **PIM** (name-level) to build distribution trees between networks.
  4. The NIC is told to accept the group's **multicast MAC** address.
- **IP → MAC mapping:** `01:00:5e` + the **low 23 bits** of the group IP (N2). 5 bits are lost, so 32 groups share one MAC. The NIC may then pass up unwanted groups, which the kernel filters out (CPU waste).
```c
int s = socket(AF_INET, SOCK_DGRAM, 0);
int one = 1; setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &one, sizeof one);
struct sockaddr_in a = { .sin_family = AF_INET, .sin_port = htons(26400),
                         .sin_addr.s_addr = inet_addr("233.54.12.1") };    // bind to the group
bind(s, (struct sockaddr *)&a, sizeof a);
struct ip_mreq m = { .imr_multiaddr.s_addr = inet_addr("233.54.12.1"),
                     .imr_interface.s_addr = inet_addr("10.1.1.5") };   // which NIC
setsockopt(s, IPPROTO_IP, IP_ADD_MEMBERSHIP, &m, sizeof m);            // ⇒ IGMP join
// then recv() in a busy loop (or read the NIC ring directly with kernel bypass)
```

### 2.3 Feed protocols, gap detection and recovery
**Message structure** (typical of exchange feeds such as ITCH/MoldUDP64-style designs):
- Each UDP packet has a header: **session ID, sequence number of its first message, message count**. Then one or more small binary messages follow: "add order 123: buy 100 @ 250.05", "execute order 123", "cancel order 77".
- These are **incremental** updates: the receiver maintains its own copy of the order book by applying every message **in sequence**. One missing message means the book is **wrong**.
- **Snapshot** channels periodically publish the full book state, so late joiners and recovering receivers can start from a known point.

**Gap detection:** keep `expected` = the next sequence number needed.
- packet seq == expected ⇒ apply it;
- seq < expected ⇒ **duplicate** (drop it);
- seq > expected ⇒ **gap**: messages expected … seq − 1 are missing.

**A/B line arbitration:** the exchange sends the **same feed twice**, on two independent networks (A and B). The handler takes **whichever copy arrives first** and drops the second as a duplicate. A loss on one line is invisible if the other line delivered that packet (N6).
```cpp
// A/B arbitration + gap detection (simplified)
uint64_t expected = 1;                                   // next sequence number needed
void on_packet(uint64_t seq, uint32_t count, const Msg* msgs) {   // from line A or B
    if (seq + count <= expected) return;                 // entirely seen already (other line won)
    if (seq > expected) {                                // GAP: something is missing on both lines
        request_recovery(expected, seq - 1);             // retransmission service or snapshot
        buffer_out_of_order(seq, count, msgs);
        return;
    }
    for (uint32_t i = expected - seq; i < count; ++i)    // skip the part already applied
        apply_to_book(msgs[i]);
    expected = seq + count;
    drain_buffered_in_order();                           // apply buffered packets that now fit
}
```
**Recovery options:**
1. **Retransmission request** to an exchange recovery server (over TCP or unicast UDP): "send messages 1001–1003".
2. **Snapshot + replay:** take the latest snapshot, then apply the buffered incremental messages after it.
3. Meanwhile, mark the instrument's book **stale** and **don't trade on it**. Trading on a wrong book is worse than not trading.

This is the "reliability by sequence numbers, not retransmission" design from note 03 §2.8.

### 2.4 Order entry: TCP sessions
- Orders go over **TCP**, usually a long-lived session per gateway, using **FIX** (tag=value text, e.g. `35=D|55=INFY|54=1|38=100|44=1520.5|`) or a faster **native binary** protocol.
- **Session layer** on top of TCP: logon, **application sequence numbers** (to detect gaps across reconnects), **heartbeats** (e.g. every 30 s; note 06 §2.10), resend requests, and exchange **throttles** (max orders/s).
- **Why TCP here:** every order must arrive **exactly once, in order**, and acknowledgements and fills come back on the same session. Lost orders are unacceptable; the overhead is acceptable because the session is pre-established.
- **TCP tuning for orders:**
  - `TCP_NODELAY` and **one `send()` per order** (note 07 §2.5);
  - connections **established before the market opens**;
  - `tcp_slow_start_after_idle = 0`, plus heartbeats so the window and caches stay warm;
  - busy polling;
  - a user-space TCP stack (OpenOnload, ef_vi, Exablaze/Cisco/AMD stacks) or an **FPGA TCP** engine for the fastest firms.

### 2.5 Where the latency goes, and how each part is cut
| Component | Typical | How firms cut it |
|---|---|---|
| **Propagation** between cities | ~5 µs/km in fibre | straighter routes; **microwave / millimetre-wave / laser** through air (~3.3 µs/km); hollow-core fibre |
| Propagation inside colo | ~5 ns per metre | colocation; short cables; exchanges equalise lengths |
| **Serialization** | 200 B: 160 ns at 10G, 64 ns at 25G | faster links, small binary messages |
| **Switching** | store-and-forward µs; cut-through ~100–300 ns; **Layer-1 switches** (pure signal replication/crosspoint, no packet parsing) ~5 ns | cut-through or L1 switches; fewer hops |
| **Host network stack** | kernel ~5 µs + jitter; bypass < 1 µs | kernel bypass, busy polling, NIC tuning (note 09) |
| **Application** | 100s of ns to µs | cache-friendly code, no syscalls/allocations on the hot path ([COA 14](../Computer%20Organization%20and%20Architecture/14-Low-Latency-Hardware-for-HFT.md)) |
| **FPGA path** | tens to hundreds of ns tick-to-trade | parse the feed and fire orders in hardware; software sets the parameters |

### 2.6 Microwave vs fibre
- Light in glass fibre travels at c / n with refractive index n ≈ 1.47 ⇒ ~2.04 × 10⁸ m/s ⇒ **~4.9 µs/km**.
- Through air, microwave signals travel at ≈ c ⇒ **~3.34 µs/km**, about **32 % less delay per km**. Fibre also follows roads and railways, while towers can follow a near-straight line. Both effects add up.
- **Trade-offs of microwave:**
  - **rain fade** (heavy rain weakens the signal, so firms keep fibre as a fallback);
  - **low bandwidth** (hundreds of Mbps), so only the most critical data is sent;
  - line of sight with towers every ~50 km, each hop adding a little latency;
  - licences and expensive towers.
- **Millimetre-wave** and free-space **laser** links carry more bandwidth over shorter distances. Shortwave radio has been tried for transatlantic routes (name-level).
- *Interview point:* this is a **propagation** problem. Upgrading 10 → 100 Gbps does nothing for it (note 01).

### 2.7 Time synchronisation: NTP, PTP and hardware timestamps
**Why exact time matters:** to measure one-way latency between machines, to order events from different feeds, to post-trade-analyse "who saw what first", and because of **regulation** (e.g. Europe's MiFID II requires HFT timestamps traceable to UTC within **100 µs**, at ≤ 1 µs granularity).

| | NTP | PTP (IEEE 1588) |
|---|---|---|
| Accuracy | ~ms over the Internet; ~tens of µs on a LAN | **sub-µs, down to tens of ns** with hardware support |
| Timestamps taken | in software (kernel/app ⇒ jitter) | in the **NIC/switch hardware** at the wire |
| Network support | none | **boundary clocks / transparent clocks** in switches correct for their own queuing delay |
| Source | NTP servers | a **grandmaster** clock disciplined by **GPS**, often with a PPS (pulse-per-second) signal |

**How PTP computes the offset** (the classic 4-timestamp exchange):
```
Master                                Slave
  t1 ── Sync (+ Follow_Up carrying t1) ──► t2       (t2 read on the slave's clock)
  t4 ◄──────────── Delay_Req ──────────── t3        (t4 read on the master's clock)
     ─────────── Delay_Resp (carrying t4) ──►

delay  = ((t2 − t1) + (t4 − t3)) / 2         (one-way path delay)
offset = ((t2 − t1) − (t4 − t3)) / 2         (slave clock − master clock)  → slave corrects by −offset
```
- **Assumption:** the path delay is **symmetric** (the same both ways). Any asymmetry a creates an error of a/2, so cable lengths and switch behaviour matter (N3).
- **Hardware timestamping:** the NIC stamps the packet when its first bit crosses the wire, using its own PTP-disciplined clock (`SO_TIMESTAMPING`). This removes the software jitter (interrupts, scheduling) from the measurement.

### 2.8 Microbursts and jitter
- **Microburst:** traffic that exceeds a link's rate for **microseconds to milliseconds**, invisible in 1-second averages (a link "at 20 % utilisation" can still drop packets). Typical causes: the market open, big news, many feeds **fanning into one switch port**.
- Excess traffic queues in the switch buffer (adding delay), and when the buffer overflows, packets drop (N5). Low-latency switches have **small buffers**, so they drop sooner.
- **Monitoring:** high-resolution counters (switch telemetry at µs granularity), packet capture appliances, per-message latency histograms.
- **Tails matter:** HFT cares about **p99 / p99.9 / max** latency, not the average. A strategy that's fast on average but occasionally takes 100 µs loses exactly the trades that matter (busy moments).

### 2.9 Measuring latency correctly
- **Passive taps / port mirroring** copy packets to a capture appliance that hardware-timestamps both the market-data packet in and the order out ⇒ the true **wire-to-wire** latency, with no observer effect on the trading host.
- Inside the host: hardware RX timestamp (NIC) → software timestamps (`rdtscp`, [OS 13 §2.8](../Operating%20Systems/13-Linux-Internals-and-Low-Latency-Tuning.md)) at each stage → hardware TX timestamp.
- Report **percentile histograms** (p50, p90, p99, p99.9, max) under realistic load, including bursts.

### 2.10 How to read this note: must know / name-only / skip
| Must know | Name-only | Skip |
|---|---|---|
| colo, tick-to-trade; why multicast (fairness, efficiency, no back-pressure, no HOL); IGMP join + snooping; multicast MAC mapping; sequenced incremental feeds + snapshots; gap detection, A/B arbitration, recovery, stale book; order entry over TCP + session seq + heartbeats + TCP tuning; latency components table; microwave vs fibre numbers + trade-offs; PTP offset formula + symmetry assumption + HW timestamps; microbursts; percentiles | PIM, L1 switches, mm-wave/laser/shortwave, MiFID II details, specific feed protocols (ITCH, MoldUDP64, SBE) | FPGA design details (Digital Logic notes), specific vendor products |

---

## 3. Worked Numericals

**N1. Chicago ↔ New Jersey: fibre vs microwave.** Straight-line distance ≈ 1,150 km. A good fibre route ≈ 1,330 km; a microwave route ≈ 1,180 km (towers almost on the straight line).
- Fibre: 1,330 km × 4.9 µs/km ≈ **6.5 ms one way** (~13 ms round trip).
- Microwave: 1,180 km × 3.34 µs/km ≈ **3.9 ms one way** (~8 ms round trip), plus a little per tower.
- Saving ≈ **2.6 ms one way**: an eternity when tick-to-trade is a few µs. The theoretical limit (vacuum, straight line) is 1,150 × 3.34 ≈ 3.84 ms.

**N2. Multicast MAC mapping.** The group 239.1.2.3.
- Low 23 bits of the IP: the second octet 1 = `00000001` → drop its top bit → `0000001`; the third octet = 2; the fourth = 3.
- MAC = `01:00:5e:01:02:03`.
- 224.1.2.3 and 239.129.2.3 (129 = `10000001` → top bit dropped → `0000001`) map to the **same MAC**. The NIC accepts all three groups and the kernel must filter. Choose group addresses that don't collide.

**N3. PTP offset and delay.** Timestamps in ns: t1 = 0 (master), t2 = 1,700 (slave), t3 = 5,000 (slave), t4 = 4,300 (master).
- t2 − t1 = 1,700; t4 − t3 = −700.
- delay = (1,700 + (−700)) / 2 = **500 ns**; offset = (1,700 − (−700)) / 2 = **1,200 ns** ⇒ the slave is 1.2 µs ahead; it subtracts 1.2 µs.
- If the forward path really takes 600 ns and the reverse 400 ns (asymmetry 200 ns), PTP still computes 500 ns each way ⇒ the offset estimate is off by **100 ns** (half the asymmetry).

**N4. A tick-to-trade budget (illustrative, software with kernel bypass, inside colo).**
| Step | Time |
|---|---|
| Market-data packet (200 B) serialization at 10G | 160 ns |
| Cable exchange → firm switch (30 m) + cut-through switch | 150 + 250 ns |
| Switch → NIC (5 m) + NIC → app memory (bypass, polling) | 25 + 700 ns |
| Feed handler parse + book update + strategy decision | 1,000 ns |
| Order build + NIC TX (bypass) | 600 ns |
| Order serialization (100 B) + switch + cable to exchange | 80 + 250 + 150 ns |
| **Total** | **≈ 3.4 µs** |
Moving the parse + decision + order path to an FPGA can cut the host part (~2.3 µs) to a few hundred ns.

**N5. A microburst at a switch port.** Four feed ports each send at 5 Gbps for **1 ms** at the open, all toward one 10 Gbps port.
- Incoming 20 Gbps vs outgoing 10 Gbps ⇒ excess 10 Gbps × 1 ms = 10⁷ bits = **1.25 MB** must be buffered.
- With a 1 MB buffer for that port ⇒ **~250 KB dropped**. The last buffered packet waits 1 MB × 8 / 10¹⁰ = **0.8 ms**: a huge delay by HFT standards. Over the whole second, average utilisation looks like a harmless ~2 %.

**N6. Value of A/B arbitration.** Each line independently loses a given packet with probability 10⁻⁶. The feed carries 2 million packets/day.
- One line only: expected gaps = 2 × 10⁶ × 10⁻⁶ = **2 per day**, each needing recovery (a stale book for ~ms).
- A/B, independent losses: P(both lose the same packet) = 10⁻¹² ⇒ expected gaps = 2 × 10⁻⁶ per day ⇒ **one every ~1,400 years**. (In reality losses are correlated, because the same burst hits both, so gaps still happen. Hence recovery paths are still needed.)

**Try it:** (a) A 600 km route: what is the one-way latency over fibre (route factor 1.15 over straight line) vs microwave (route factor 1.02)? (b) PTP: t1 = 10,000, t2 = 10,900, t3 = 12,000, t4 = 12,500 (ns). What are the delay and offset?
<details><summary>Answer</summary>(a) Fibre: 690 km × 4.9 ≈ <b>3.38 ms</b>; microwave: 612 km × 3.34 ≈ <b>2.04 ms</b>. (b) t2 − t1 = 900, t4 − t3 = 500 ⇒ delay = <b>700 ns</b>, offset = <b>200 ns</b> (slave ahead).</details>

---

## 4. Interview Questions

### Basic
- **Q: Why do exchanges publish market data over UDP multicast?** Fairness (one simultaneous copy to everyone), efficiency, no back-pressure from slow receivers, and no head-of-line blocking from retransmissions.
- **Q: How does a server start receiving a multicast group?** `IP_ADD_MEMBERSHIP` ⇒ the kernel sends an IGMP report ⇒ snooping switches forward the group to that port; the NIC accepts the group's MAC.
- **Q: What is tick-to-trade latency?** The time from the market-data packet reaching your NIC to your order leaving it. Software with bypass ~1–5 µs; FPGA ~100s of ns.

### Intermediate
- **Q: How do you detect and handle a lost market-data packet?** Sequence numbers ⇒ gap detection. First rely on A/B arbitration; otherwise request a retransmission or use a snapshot + replay. Mark the book stale and don't trade until it's rebuilt.
- **Q: What is A/B line arbitration?** The same feed on two independent networks; take the first copy of each sequence number and drop the duplicate. Loss on one line is hidden (N6).
- **Q: Why is order entry TCP when market data is UDP?** Orders need exactly-once in-order delivery and acknowledgements over a long-lived session; the setup cost is paid once. Tune with NODELAY, a single send per message, warm connections, and kernel-bypass TCP.
- **Q: Microwave vs fibre?** Air ≈ c (3.34 µs/km) vs glass ≈ c/1.47 (4.9 µs/km), plus straighter paths ⇒ ~30–40 % lower latency. Costs: rain fade, low bandwidth, towers, licences; fibre kept as backup.
- **Q: What is a microburst and why can a lightly loaded link drop packets?** A ms/µs-scale burst above line rate (fan-in at the open) overflows small switch buffers; 1-second averages hide it (N5).

### Advanced / killer follow-ups
- **Q [HFT]: Design a feed handler.**
  - Receive with kernel bypass and busy polling on an isolated, NUMA-local core.
  - Run A/B arbitration by sequence number, then gap detection → recovery (TCP retransmission / snapshot) with a stale flag.
  - Parse binary messages with zero copy.
  - Keep the order book per instrument in pre-allocated, cache-friendly arrays (price-level array or hash + intrusive lists).
  - Publish updates to strategy threads through a lock-free single-producer/single-consumer ring.
  - Hardware RX timestamps for latency monitoring; no allocations, syscalls or locks on the hot path.
- **Q [HFT]: How does PTP synchronise clocks? What limits its accuracy?** The 4-timestamp exchange gives delay and offset (§2.7). Limits: path asymmetry (error = half of it), timestamping point (needs hardware), switch queuing (boundary/transparent clocks fix it), oscillator stability.
- **Q [HFT]: Why prefer hardware timestamps over `clock_gettime` in the app?** Software stamps include interrupt, softirq and scheduling jitter (µs). NIC stamps record the actual wire time (ns), consistent across hosts via PTP.
- **Q: Your strategy's p50 latency is great but p99.9 is 50× worse. Where do you look?** Microbursts and queuing at switches/NIC rings, interrupts or softirqs on the trading core, page faults, cache/TLB misses after idle, logging, SMIs, power states, GC. Measure with per-stage timestamps; fix with isolation, pre-faulting, busy polling ([OS 13 §2.6](../Operating%20Systems/13-Linux-Internals-and-Low-Latency-Tuning.md)).
- **Q: Two multicast groups share a MAC address. What problem does that cause?** The NIC accepts both (only 23 bits are mapped), so the kernel or app processes and drops unwanted traffic: wasted CPU and cache pollution on the hot core. Pick non-colliding groups or filter in hardware.
- **Q: Why might an exchange add a deliberate delay ("speed bump")?** To blunt pure-latency races and protect resting orders (e.g. IEX's 350 µs fibre coil). It's a market-design question, but it shows awareness.

---

## 5. Common Traps
- Proposing TCP for market-data distribution, or inline retransmission that blocks later messages.
- Forgetting the stale-book state during recovery (trading on a gapped book).
- Thinking more bandwidth reduces inter-city latency (it's propagation).
- Quoting average latency instead of tail percentiles.
- Assuming PTP is accurate without hardware timestamps or with asymmetric paths.
- Ignoring microbursts because average utilisation is low.
- Forgetting that multicast MAC mapping is 23 bits (32 groups per MAC).

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Setup | colo; market data out = UDP multicast (A/B); orders in = TCP (FIX/binary) |
| Tick-to-trade | NIC in → NIC out; ~1–5 µs software bypass, ~100s ns FPGA |
| Multicast | 224/4; IGMP join; snooping switches; MAC 01:00:5e + low 23 bits |
| Feeds | sequenced incremental messages + snapshots; book must apply every message |
| Gaps | expected seq; duplicates dropped; gap ⇒ A/B, retransmit request, snapshot + replay; stale flag |
| Orders | TCP session seq + heartbeats + throttles; NODELAY, 1 send/msg, warm, no idle slow start, bypass/FPGA TCP |
| Latency parts | propagation (µs/km) · serialization (L/R) · switch (L1 ~5 ns, cut-through ~100s ns) · stack · app |
| Fibre vs microwave | 4.9 vs 3.34 µs/km; Chicago–NJ ~6.5 vs ~3.9 ms one way; rain, bandwidth |
| PTP | delay = ((t2−t1)+(t4−t3))/2, offset = ((t2−t1)−(t4−t3))/2; symmetric path; HW timestamps; GPS grandmaster |
| Microbursts | fan-in > line rate for µs–ms ⇒ queue/drop; invisible in averages |
| Measure | taps + HW timestamps; percentiles (p99.9, max) |
