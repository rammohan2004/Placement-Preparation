# 07. TCP Reliability, Flow & Congestion Control

> **Asked at:** [AMZN] [DESHAW] flow vs congestion control, slow start / congestion avoidance / fast recovery, cwnd traces, why throughput ≈ window/RTT, CUBIC vs BBR, head-of-line blocking · [HFT] Nagle + delayed ACK (the 40 ms stall), `TCP_NODELAY`, `TCP_QUICKACK`, slow start after idle, loss recovery latency · [NVIDIA] window sizing for high-BDP links, incast in clusters, why RDMA avoids TCP · [QCOM] [APPLE] TCP over lossy wireless (loss ≠ congestion).
> **Prerequisites:** [01](01-Network-Basics-and-Layered-Models.md) (RTT, bandwidth-delay product, queues and loss), [03](03-Reliable-Data-Transfer-and-Sliding-Window.md) (windows, cumulative ACKs, timeouts), [06](06-Transport-Layer-UDP-and-TCP-Connections.md) (TCP header, seq/ack, handshake).

---

## 1. Intuition
A TCP sender is a car on a highway it can't see, with a passenger it must not overwhelm.
- **Flow control** is the passenger saying "slow down, I can't keep up". The **receiver** advertises how much buffer it has left (`rwnd`).
- **Congestion control** is the sender guessing the **traffic on the road**. It speeds up gently while all is well and **brakes hard when it sees a crash** (packet loss). The guess is the congestion window (`cwnd`).
- **Reliability** means that if a parcel goes missing, the sender notices (timeout or duplicate ACKs) and sends it again.

The sender may have at most **min(rwnd, cwnd)** bytes unacknowledged. So throughput ≈ window / RTT.

---

## 2. Deep Dive

### 2.1 Reliability: how TCP detects and repairs loss
From note 03, TCP is a hybrid of Go-Back-N and Selective Repeat. Its pieces:
| Mechanism | What it does |
|---|---|
| **Byte sequence numbers + cumulative ACK** | ACK = next byte expected ("I have everything before N") |
| **Receiver buffers out-of-order data** | segments after a gap are kept, not discarded (unlike GBN) |
| **Retransmission timeout (RTO)** | if the oldest unacknowledged segment isn't ACKed within the RTO ⇒ retransmit it |
| **Fast retransmit** | **3 duplicate ACKs** ⇒ retransmit the missing segment **immediately**, without waiting for the RTO |
| **SACK** (selective ACK option) | the receiver lists the byte ranges it holds beyond the gap ⇒ the sender resends only the holes |

**Duplicate ACKs, step by step:** segment 3 is lost; segments 4, 5 and 6 arrive.
```
Sender sends:  1  2  3(lost)  4  5  6
Receiver ACKs: ACK2 ACK3  ACK3  ACK3  ACK3      ← "still waiting for 3" each time
                          dup1  dup2  dup3 ⇒ sender fast-retransmits 3
Receiver gets 3 ⇒ it already holds 4,5,6 ⇒ ACK7 (jumps forward)
```
- **Why 3 duplicates and not 1:** packets can be **reordered** slightly, which produces 1–2 duplicate ACKs without any loss. Three is the conventional "it's really lost" threshold.
- **RTO calculation** (RFC 6298). After each RTT sample R (measured with timestamps):
```
SRTT   = (1 − 1/8)·SRTT + 1/8 · R                (smoothed RTT)
RTTVAR = (1 − 1/4)·RTTVAR + 1/4 · |SRTT − R|     (variation)
RTO    = SRTT + 4 · RTTVAR                        (Linux minimum 200 ms; RFC says 1 s)
```
  On each timeout the RTO **doubles** (exponential backoff). **Karn's rule:** never take an RTT sample from a retransmitted segment, because it's ambiguous which copy the ACK is for. Timestamps solve that.
- **Cost:** fast retransmit costs ~1 RTT. A **timeout costs ≥ 200 ms on Linux** (often much more), which is catastrophic for latency. Tail losses (the *last* packets of a burst, which produce no duplicate ACKs) are the worst case. **RACK-TLP** (Linux's modern loss detection) sends a "tail loss probe" to trigger SACK/duplicate ACKs quickly.

### 2.2 Flow control: don't overrun the receiver
- The receiver's kernel has a **receive buffer** per socket. Data waits there until the application calls `recv()`.
- Every ACK carries **rwnd** (receive window) = free space in that buffer. The sender keeps **unacknowledged bytes ≤ rwnd**.
- If the app stops reading, the buffer fills, rwnd drops to **0** ("zero window"), and the sender stops. It sends occasional **window probes** until space opens. *This is how a slow consumer back-pressures a fast producer.*
- **Window scaling:** the header's window field is 16 bits, so max **64 KB**, far below the BDP of modern links (note 01 N4). The **window-scale option** (negotiated in the SYNs) multiplies it by 2^shift (shift ≤ 14) ⇒ up to **1 GB**. Linux **auto-tunes** buffer sizes (`net.ipv4.tcp_rmem`/`tcp_wmem`).
- **Throughput limit:** throughput ≤ rwnd / RTT. A 64 KB window over 100 ms RTT ⇒ ≤ 5.2 Mbps, whatever the link speed.

### 2.3 Congestion control: don't overrun the network
**Why it's needed:** if every sender transmits as fast as its receiver allows, router queues overflow, packets drop, senders retransmit, and the network carries mostly retransmissions. This is **congestion collapse**; it actually happened on the Internet in 1986. TCP must therefore **infer** the network's spare capacity and share it fairly, **with no help from routers** (classic TCP).

**The signal:** classic TCP treats **packet loss** as "the network is congested" (queues overflowed). Newer schemes also use **delay** (RTT rising ⇒ queues building) or **ECN marks**.

**The state:** the sender keeps **cwnd** (congestion window, in bytes or segments) and **ssthresh** (slow-start threshold).
```
bytes in flight ≤ min(cwnd, rwnd)        throughput ≈ min(cwnd, rwnd) / RTT
```

**The algorithm (TCP Reno; the textbook answer):**
| Phase | Rule | Growth |
|---|---|---|
| **Slow start** (cwnd < ssthresh) | cwnd += 1 MSS **per ACK** | **doubles every RTT** (exponential), despite the name |
| **Congestion avoidance** (cwnd ≥ ssthresh) | cwnd += 1 MSS **per RTT** (MSS × MSS/cwnd per ACK) | linear: **Additive Increase** |
| **3 duplicate ACKs** (mild congestion: packets still flow) | ssthresh = cwnd/2; cwnd = ssthresh (+3); **fast recovery**; then congestion avoidance | **Multiplicative Decrease** (halve) |
| **Timeout** (severe: nothing getting through) | ssthresh = cwnd/2; **cwnd = 1 MSS**; slow start again | back to the start |

- **Initial cwnd:** 10 MSS on Linux (≈ 14.6 KB, RFC 6928; older stacks used 1–4).
- **AIMD** (Additive Increase, Multiplicative Decrease) gives the famous **sawtooth** and is provably **fair**: two flows sharing a link converge to equal shares, because halving shrinks the larger flow more in absolute terms.
```
cwnd
 ▲          3 dup ACKs                         timeout
 │           ╱│      ╱│                          │
 │  slow    ╱ │     ╱ │   ╱│                     │
 │  start  ╱  │    ╱  │  ╱ │                     │
 │       ╱    └──╱    └─╱  └───────╱│            │
 │     ╱    (halve, linear growth)  │            └─ cwnd = 1, slow start again
 │   ╱                               ssthresh ↓
 │ ╱
 └──────────────────────────────────────────────────► time (RTTs)
```
- **TCP Tahoe** (older): cwnd = 1 on **any** loss (no fast recovery). **Reno**: fast recovery on 3 duplicate ACKs. **NewReno**: handles several losses in one window better. **SACK-based** recovery: modern default behaviour.

### 2.4 Modern congestion control: CUBIC and BBR
| | **Reno** | **CUBIC** (Linux/Windows/macOS default) | **BBR** (Google; available in Linux) |
|---|---|---|---|
| Congestion signal | loss | loss | **measured bandwidth + min RTT** (a model) |
| Growth after a loss | +1 MSS per RTT | a **cubic function of time since the last loss**: fast far from the old maximum, cautious near it, then probes beyond | paces at the estimated bottleneck bandwidth; periodically probes for more bandwidth and for a lower RTT |
| On loss | halve | cut to ~0.7 × cwnd | doesn't treat loss as the main signal |
| Strength | simple, fair | fills **high-BDP links** much faster than Reno; RTT-fair-ish | high throughput on lossy links; keeps queues short (**low latency**) |
| Weakness | slow on long fat pipes | fills buffers before backing off ⇒ **bufferbloat** | fairness issues against CUBIC; more complex |

- **Long fat network** = high bandwidth × high RTT. Reno at 10 Gbps / 100 ms would need ~1 hour of linear growth to refill its window after one loss (N3), which is why CUBIC exists.
- **Bufferbloat:** oversized router/modem buffers let loss-based TCP fill them completely ⇒ seconds of queuing delay (your video call stutters while a download runs). Fixes: AQM in routers (**CoDel**, **fq_codel**: drop or mark early when the queueing delay grows), **ECN**, and BBR.
- **ECN** (Explicit Congestion Notification): a router **marks** the IP header's ECN bits instead of dropping. The receiver echoes it (ECE flag), and the sender reduces cwnd as if a loss happened, **without losing the packet**. Data centres use this heavily (**DCTCP**: cuts cwnd in proportion to the fraction of marked packets ⇒ very shallow queues).
- **Pacing:** instead of sending a window as a burst, spread packets evenly over the RTT (Linux `fq` qdisc). Fewer bursts mean fewer drops and smaller queues. BBR requires it.

### 2.5 Small messages: Nagle's algorithm, delayed ACK and the 40 ms stall [HFT]
**Nagle's algorithm** (on by default) avoids flooding the network with tiny packets (e.g. one keystroke = 1 byte + 40 bytes of headers):
> If there is **unacknowledged data in flight**, buffer new small writes until either an ACK arrives or a full MSS accumulates.

**Delayed ACK** (on by default on the receiver): don't ACK every segment immediately. Wait up to ~40 ms (Linux minimum) to 200 ms, hoping to piggyback the ACK on response data or ACK two segments at once.

**Together they deadlock briefly:**
```
Client: write(header, 100 B) → sent immediately (nothing in flight)
Client: write(body, 50 B)    → Nagle: data is in flight, so HOLD until the ACK comes
Server: got 100 B, but its app waits for the full request → no response to piggyback on
        → delayed ACK timer: waits ~40 ms before sending a bare ACK
Client: ACK arrives after ~40 ms → body finally sent
⇒ every request takes +40 ms (or +200 ms on some stacks)
```
**Fixes:**
- **`TCP_NODELAY`**: disables Nagle; small writes go out immediately. **Every latency-sensitive app sets it** (trading gateways, RPC frameworks, Redis clients, games).
- Better still, **write the whole message in one call** (`writev()` for header + body), so there's nothing to hold back.
- `TCP_QUICKACK` (Linux) temporarily disables delayed ACKs on the receiver. It must be re-armed, because the kernel may switch back.
- `TCP_CORK` is the opposite tool: intentionally accumulate data, then uncork to send full segments (file servers).

### 2.6 Other latency effects you should know
- **Head-of-line (HOL) blocking:** TCP delivers bytes **in order**. If segment 3 is lost, segments 4–100 sit in the receive buffer, **unavailable to the app**, until 3 is retransmitted (≥ 1 RTT). Every message multiplexed on that connection stalls. This motivates HTTP/3 over **QUIC**, which has independent streams (note 08), and UDP for market data.
- **Slow start after idle:** after an idle period longer than the RTO, Linux resets cwnd back to the initial window (`net.ipv4.tcp_slow_start_after_idle = 1`). A burst after a quiet period, such as a market open, is then throttled. Low-latency servers set it to 0.
- **Short flows never leave slow start:** a 50 KB response with IW = 10 segments (14.6 KB) needs ~3 RTTs (14.6 → 29 → 58 KB). Latency is set by RTT and the initial window, not bandwidth (N2).
- **Incast** (data centres) [NVIDIA]: many servers answer one client simultaneously (scatter-gather, distributed storage, ML parameter reads). The switch port's buffer overflows and several flows hit timeouts (≥ 200 ms), so throughput collapses. Mitigations: ECN/DCTCP, smaller minimum RTO, staggering the requests, and RDMA with lossless fabrics (note 11).
- **Wireless loss ≠ congestion** [QCOM] [APPLE]: Wi-Fi/LTE drop packets because of radio noise, and loss-based TCP wrongly halves cwnd. Link-layer retransmission (note 02) hides most of this; BBR and QUIC cope better.

### 2.7 Throughput formulas
- **Window-limited:** throughput ≤ min(cwnd, rwnd) / RTT.
- **Loss-limited (Mathis formula, Reno-style):** throughput ≈ **(MSS / RTT) × (1.22 / √p)**, where p = loss rate. Throughput falls with RTT and with √loss. Two flows with different RTTs on the same bottleneck get unequal shares (the short-RTT flow wins), which is classic TCP's **RTT unfairness**.

### 2.8 Tools
- `ss -tni`: per-connection **cwnd, rtt, rto, retransmits, pacing rate, delivery rate**. The best quick look.
- `sysctl net.ipv4.tcp_congestion_control` (cubic/bbr); `sysctl net.ipv4.tcp_available_congestion_control`.
- `tcpdump`/Wireshark (watch for "TCP Dup ACK", "Retransmission", "Zero Window"); `iperf3` for throughput tests; `nstat -az | grep -i retrans`.

### 2.9 How to read this note: must know / name-only / skip
| Must know | Name-only | Skip |
|---|---|---|
| cumulative ACK + buffering; RTO vs 3 dup ACKs; SACK; RTO formula idea + backoff; rwnd + zero window + window scaling; why congestion control; cwnd/ssthresh; slow start, AIMD, fast recovery vs timeout; sawtooth + fairness; window ÷ RTT; Nagle + delayed ACK + TCP_NODELAY; HOL blocking; slow start after idle; bufferbloat/ECN idea; CUBIC vs BBR one-liners | Tahoe/NewReno details; RACK-TLP; DCTCP; Mathis formula constant; pacing internals | the CUBIC equation's constants, BBR state machine details, Vegas/Westwood/Compound variants |

---

## 3. Worked Numericals

**N1. cwnd trace (Reno), in MSS units.** Initial cwnd = 1, ssthresh = 16. No loss until cwnd = 20, then **3 duplicate ACKs**; later, at cwnd = 14, a **timeout**.
| RTT | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | … |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| cwnd | 1 | 2 | 4 | 8 | **16** | 17 | 18 | 19 | **20** | 10 | 11 | 12 | 13 | 14 |
- RTT 1–5: slow start doubles 1 → 16 (reaching ssthresh). RTT 5 onward: +1 per RTT.
- At 20, 3 duplicate ACKs ⇒ ssthresh = 10, cwnd = 10 (fast recovery) ⇒ linear again.
- At 14, a **timeout** ⇒ ssthresh = 7, cwnd = **1**. Slow start: 1, 2, 4, then 7 (capped at ssthresh), then 8, 9, … linearly.

**N2. Short transfer latency (slow start dominates).** Download 100 KB; MSS = 1460 B; initial window = 10 MSS; RTT = 80 ms; plenty of bandwidth; the connection is already open.
- Data per RTT: 10, 20, 40, 80 segments ⇒ cumulative 14.6 KB, 43.8 KB, 102 KB.
- 100 KB needs **3 RTTs ≈ 240 ms** (+ the request's own half-RTT). At 1 Gbps the raw transmit time would be 0.8 ms; **latency is RTT-bound**. Add 1 RTT for a new TCP handshake and 1 for TLS 1.3 ⇒ ~400 ms.

**N3. Why Reno fails on long fat pipes.** 10 Gbps, RTT 100 ms, MSS 1500 B.
- Window needed = BDP = 10¹⁰ × 0.1 / (1500 × 8) ≈ **83,333 segments**.
- After one loss, Reno halves to ~41,667 and grows by 1 segment per RTT ⇒ 41,667 RTTs × 0.1 s ≈ **4167 s ≈ 70 minutes** to refill the link.
- Mathis: to sustain 10 Gbps it would need a loss rate p ≤ (1.22 × MSS / (RTT × throughput))² = (1.22 × 12,000 / (0.1 × 10¹⁰))² ≈ **2 × 10⁻¹⁰**, i.e. at most one loss per ~5 billion packets. Unrealistic ⇒ CUBIC/BBR.

**N4. Flow-control limit.** The receiver advertises rwnd = 64 KB (no window scaling); RTT = 30 ms; the link is 1 Gbps.
- Max throughput = 65,536 × 8 / 0.03 ≈ **17.5 Mbps** (1.7 % of the link).
- For 1 Gbps: rwnd ≥ 10⁹ × 0.03 / 8 = **3.75 MB** ⇒ needs window scaling (shift ≥ 6, since 64 KB × 2⁶ = 4 MB).

**N5. RTO computation.** SRTT = 100 ms, RTTVAR = 20 ms. A new sample R = 140 ms.
- RTTVAR = 0.75 × 20 + 0.25 × |100 − 140| = 15 + 10 = **25 ms** (using the old SRTT).
- SRTT = 0.875 × 100 + 0.125 × 140 = 87.5 + 17.5 = **105 ms**.
- RTO = 105 + 4 × 25 = **205 ms**. If it times out: 410 ms, then 820 ms, … (exponential backoff).

**N6. [HFT] Nagle + delayed ACK cost.** A gateway sends each order as two `write()`s (header 40 B, body 60 B) on a socket **without** `TCP_NODELAY`; the exchange's delayed-ACK timer is 40 ms; the network RTT is 50 µs.
- The first write is sent at once; the second waits for an ACK, which comes only when the delayed-ACK timer fires ⇒ **~40 ms per order** instead of ~25 µs one way, a factor of **1600×**.
- Fix: `TCP_NODELAY` **and** a single `writev()`/`send()` per message.

**N7. Loss-limited throughput (Mathis).** MSS 1460 B, RTT 50 ms, loss rate p = 10⁻⁴.
- Throughput ≈ (1460 × 8 / 0.05) × 1.22 / √10⁻⁴ = 233,600 × 1.22 / 0.01 ≈ **28.5 Mbps**, whatever the link speed. Halving the RTT (a closer server or CDN) doubles it.

**Try it:** (a) cwnd starts at 1 MSS, ssthresh = 8. Give cwnd for RTTs 1–8 if a timeout happens during RTT 6 (cwnd was 10 then). (b) With RTT 200 ms and a fixed window of 1 MB, what is the max throughput?
<details><summary>Answer</summary>(a) 1, 2, 4, 8, 9, 10 → timeout ⇒ ssthresh = 5, cwnd = 1 ⇒ RTT 7: 1, RTT 8: 2 (then 4, 5, 6, …). (b) 1 MB × 8 / 0.2 s ≈ <b>41.9 Mbps</b> (2²⁰ B) ≈ 40 Mbps (10⁶ B).</details>

---

## 4. Interview Questions

### Basic
- **Q: Flow control vs congestion control?** Flow: protect the **receiver's buffer** (rwnd advertised in every ACK). Congestion: protect the **network** (cwnd inferred by the sender from loss/delay). The sender uses min(rwnd, cwnd).
- **Q: How does TCP detect loss?** A timeout (RTO expires) or 3 duplicate ACKs (fast retransmit); SACK pinpoints the holes.
- **Q: What is slow start?** cwnd starts small (10 MSS) and doubles every RTT until ssthresh or a loss. It's "slow" only compared with blasting the whole window at once.

### Intermediate
- **Q: Explain AIMD and why it's fair.** +1 MSS per RTT; halve on loss. Halving cuts the bigger flow more in absolute terms, so competing flows converge to equal shares (the sawtooth).
- **Q: What happens on 3 duplicate ACKs vs a timeout?** Duplicate ACKs (mild): fast retransmit, ssthresh = cwnd/2, cwnd = ssthresh (fast recovery). Timeout (severe): ssthresh = cwnd/2, cwnd = 1, slow start again, RTO doubled.
- **Q: Why 3 duplicate ACKs?** Reordering produces 1–2 duplicates without loss; 3 is strong evidence of loss.
- **Q: What is window scaling and why is it needed?** The 16-bit window caps at 64 KB < BDP. Scaling (shift ≤ 14) allows up to 1 GB (N4).
- **Q: CUBIC vs BBR?** CUBIC: loss-based, cubic growth, fills buffers (bufferbloat), Linux default. BBR: models bottleneck bandwidth + min RTT, paces, keeps queues low, handles random loss well, but has fairness concerns.
- **Q: What is head-of-line blocking in TCP?** In-order delivery: one lost segment holds back all later data on the connection until it's retransmitted ⇒ QUIC streams fix this.

### Advanced / killer follow-ups
- **Q [HFT]: Our small RPCs take exactly ~40 ms extra. Why?** The Nagle + delayed ACK interaction (§2.5, N6). Set `TCP_NODELAY`, write each message in one call, and consider `TCP_QUICKACK` on the receiver.
- **Q [HFT]: The first orders after a quiet period are slower. Why?** Slow start after idle resets cwnd, and the cold caches/TLB of the code path add to it. Disable `tcp_slow_start_after_idle` and keep the path warm with heartbeats.
- **Q: How would you speed up a 100 KB API response for global users?** It's RTT-bound (N2): use a CDN/edge to cut RTT, reuse connections (skip handshakes), TLS 1.3/QUIC (fewer RTTs), and a larger initial window. Bandwidth won't help.
- **Q [NVIDIA]: What is TCP incast and how is it mitigated?** Synchronized responses overflow a switch buffer ⇒ several flows hit RTO timeouts ⇒ throughput collapse. Mitigations: ECN/DCTCP, a small RTO minimum, request staggering, and RDMA over lossless fabrics.
- **Q: What is bufferbloat and how do you fix it?** Oversized buffers + loss-based TCP ⇒ standing queues of 100s of ms. Fix: AQM (CoDel/fq_codel), ECN, BBR, pacing.
- **Q: Why might TCP perform badly on Wi-Fi or cellular?** Random radio loss is misread as congestion ⇒ cwnd halved unnecessarily, and variable RTT inflates the RTO. Link-layer ARQ, BBR and QUIC help.
- **Q: Two flows share a bottleneck; one has a 10 ms RTT and the other 100 ms. Who gets more bandwidth?** The 10 ms flow (Reno/CUBIC-style RTT unfairness; Mathis ∝ 1/RTT), because its window grows 10× more often.

---

## 5. Common Traps
- Calling rwnd the congestion window. rwnd = receiver buffer; cwnd = network estimate.
- Saying slow start grows linearly. It's exponential (per ACK ⇒ doubles per RTT).
- Resetting cwnd to 1 on 3 duplicate ACKs (that's a timeout; Tahoe did it for both).
- Forgetting ssthresh = cwnd/2 is set **before** cwnd changes.
- Thinking more bandwidth fixes short-flow latency (it's RTT and slow-start bound).
- Disabling Nagle but still sending each message in several writes.
- Forgetting Karn's rule (no RTT samples from retransmissions without timestamps).

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Loss detection | RTO timeout (≥ 200 ms Linux, doubles) · 3 dup ACKs ⇒ fast retransmit · SACK holes |
| RTO | SRTT + 4·RTTVAR (α 1/8, β 1/4); Karn's rule |
| Flow control | rwnd = free receive buffer; zero window + probes; window scale ⇒ 1 GB |
| In flight | ≤ min(cwnd, rwnd); throughput ≈ window / RTT |
| Slow start | cwnd += 1 MSS/ACK ⇒ ×2 per RTT; IW = 10 MSS |
| Cong. avoidance | +1 MSS per RTT (AI) |
| 3 dup ACKs | ssthresh = cwnd/2, cwnd = ssthresh (fast recovery) (MD) |
| Timeout | ssthresh = cwnd/2, cwnd = 1, slow start |
| AIMD | sawtooth; fair; RTT-unfair across different RTTs |
| CUBIC / BBR | loss-based cubic (default) / bandwidth + min-RTT model, pacing, low queues |
| Bufferbloat | big buffers ⇒ big delay; CoDel, ECN, BBR |
| Nagle + delayed ACK | +40 ms stalls ⇒ TCP_NODELAY + one write per message |
| HOL blocking | one loss stalls the whole stream ⇒ QUIC, UDP |
| Idle | slow start after idle ⇒ disable for latency |
| Formulas | Mathis ≈ (MSS/RTT)·1.22/√p |
