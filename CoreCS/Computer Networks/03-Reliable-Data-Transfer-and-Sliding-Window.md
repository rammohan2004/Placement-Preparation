# 03. Reliable Data Transfer & Sliding Window

> **Asked at:** [AMZN] [DESHAW] stop-and-wait vs sliding window, Go-Back-N vs Selective Repeat, window-size / sequence-number numericals, "design a reliable protocol on top of UDP" · [HFT] why market data is *not* retransmitted inline, gap detection with sequence numbers · [QCOM] link-layer ARQ in Wi-Fi/LTE · [NVIDIA] in-flight data and windows on high-bandwidth links.
> **Prerequisites:** [01 Network Basics](01-Network-Basics-and-Layered-Models.md) (transmission/propagation delay, RTT, bandwidth-delay product), [02 Data Link](02-Data-Link-Layer-and-Ethernet.md) (checksums/CRC detect corruption).

---

## 1. Intuition
You're dictating a phone number over a bad line.
- **Stop-and-wait:** say one digit, wait for "got it", then say the next. Safe, but painfully slow when the line has a long delay.
- **Sliding window:** say up to N digits without waiting. The listener confirms as they go, and you repeat whatever they missed.
  - **Go-Back-N:** if they missed digit 3, you repeat from 3 onwards.
  - **Selective Repeat:** you repeat only digit 3.

The mechanisms in this note (sequence numbers, ACKs, timers, windows) are exactly what TCP is built from (note 07).

---

## 2. Deep Dive

### 2.1 What can go wrong, and the tool for each problem
The network below a protocol (IP, or a radio link) is **unreliable**. A packet may be:
| Problem | Tool that fixes it |
|---|---|
| **corrupted** (bits flipped) | **checksum/CRC** to detect it (note 02); the receiver drops it |
| **lost** (dropped by a full router queue) | **timer** + **retransmission**: no acknowledgement in time ⇒ send again |
| **duplicated** (a retransmission arrives, but so does the original) | **sequence numbers** let the receiver recognise and discard copies |
| **reordered** (packets take different paths) | **sequence numbers** let the receiver put them back in order |

Definitions used below:
- **ACK** (acknowledgement): a small message from the receiver saying "I got packet n". A **NAK** (negative ACK) says "packet n was bad". Most real protocols use only ACKs plus timeouts.
- **Cumulative ACK n**: "I have received **everything up to** n" (Go-Back-N style). In TCP, ACK n means "everything before byte n; send n next".
- **Timeout**: how long the sender waits for an ACK before retransmitting. It must be a bit **longer than the RTT**. Too short ⇒ needless duplicate sends; too long ⇒ slow recovery.
- **ARQ** (Automatic Repeat reQuest): the general name for "detect loss, then retransmit" protocols. Stop-and-wait, Go-Back-N and Selective Repeat are the three classic ARQs.

Two delays from note 01:
- **T_t** = transmission time of one data packet = L / R;
- **T_p** = one-way propagation delay.

The ratio **a = T_p / T_t** says how many packets "fit" in the pipe one way. Large a (long or fast links) is where stop-and-wait collapses.

### 2.2 Stop-and-wait
Send one packet, start a timer, and wait for its ACK before sending the next.
```
Sender                                      Receiver
  |── pkt 0 ──────────────────────────────►|   T_t to push it out, T_p to arrive
  |                                         |
  |◄────────────────────────────── ACK 0 ──|   T_p back (ACK assumed tiny)
  |── pkt 1 ──────────────────────────────►|
  |        ✗ lost                           |
  |   (timer expires)                       |
  |── pkt 1 (retransmit) ─────────────────►|
  |◄────────────────────────────── ACK 1 ──|
```
- **Why sequence numbers even here:** suppose ACK 0 is lost. The sender times out and resends pkt 0. Without a number, the receiver can't tell "new packet" from "copy of the old one". **1 bit (alternating 0, 1, 0, 1)** is enough, because only one packet is outstanding at a time. Hence the name *alternating-bit protocol*.
- **Link utilisation** (fraction of time the sender is actually transmitting):
```
U = T_t / (T_t + 2·T_p) = 1 / (1 + 2a)
```
  (ignoring ACK transmission time and processing; state that assumption in answers). For a = 10, U = 1/21 ≈ **4.8 %**: the link is idle 95 % of the time.

### 2.3 Pipelining: keep N packets in flight
Allow up to **N unacknowledged packets** ("in flight") at once. N is the **window size**.
```
U = N / (1 + 2a)      (capped at 1, i.e. 100 %)
Full utilisation needs  N ≥ 1 + 2a   (the window must cover one round trip of packets)
```
This is the bandwidth-delay product from note 01, counted in packets: N × L ≥ R × RTT.

Sequence numbers now need **k bits**, so they run 0 … 2ᵏ − 1 and then **wrap around** (mod 2ᵏ). How big N may be for a given k depends on the protocol (§2.6).

**Sliding window** picture (sender side, N = 4):
```
 seq:  0   1   2   3   4   5   6   7   8   9 ...
      [ACKed ][  sent, not ACKed  ][usable][ not allowed yet ]
               └──────── window of N = 4 ───────┘
 When the oldest packet in the window is ACKed, the window slides right by one.
```

### 2.4 Go-Back-N (GBN)
**Sender:** up to N unacknowledged packets; **one timer**, for the oldest unacknowledged packet. On timeout, **resend every packet in the window** ("go back" to the oldest).
**Receiver:** window of size **1**. It accepts **only the next expected** sequence number. Out-of-order packets are **discarded**, and it re-ACKs the last in-order packet (a **cumulative ACK**).
```
Sender (N = 4)                           Receiver (expects 0)
  0 ─────────────────────────────────►   accept 0, ACK 0
  1 ─────────────────────────────────►   accept 1, ACK 1
  2 ──── ✗ lost
  3 ─────────────────────────────────►   expected 2, got 3 → DISCARD, ACK 1 again
  4 ─────────────────────────────────►   DISCARD, ACK 1
  5 ─────────────────────────────────►   DISCARD, ACK 1
  (timer for 2 expires)
  2 3 4 5 (all resent) ───────────────►   accept 2,3,4,5 in order
```
- ➕ Very simple receiver: no buffering, one expected number. One cumulative ACK can replace lost earlier ACKs.
- ➖ One loss causes up to **N retransmissions**. That's wasteful when N is large or loss is frequent.
- **Window limit: N ≤ 2ᵏ − 1** (N5 shows why).

### 2.5 Selective Repeat (SR)
**Sender:** up to N unacknowledged packets, a **timer per packet**, and it resends **only the packet whose timer expires**.
**Receiver:** window of size **N**. It **buffers** out-of-order packets that fall inside its window, **ACKs each packet individually**, and delivers to the application in order once the gap is filled.
```
Sender (N = 4)                           Receiver window [0..3]
  0 ───────────────────────────────►    ACK 0, deliver 0          window → [1..4]
  1 ───────────────────────────────►    ACK 1, deliver 1          window → [2..5]
  2 ──── ✗ lost
  3 ───────────────────────────────►    buffer 3, ACK 3
  4 ───────────────────────────────►    buffer 4, ACK 4
  5 ───────────────────────────────►    buffer 5, ACK 5
  (timer for 2 expires) 2 ──────────►   ACK 2, deliver 2,3,4,5    window → [6..9]
```
- ➕ Only the lost packet is resent. Best on high-loss or high-BDP links.
- ➖ Receiver buffers and per-packet timers; more complex.
- **Window limit: N ≤ 2ᵏ⁻¹** (half the sequence space; N5).

### 2.6 Comparison, and why the window limits exist
| | Stop-and-wait | Go-Back-N | Selective Repeat |
|---|---|---|---|
| Sender window | 1 | N | N |
| Receiver window | 1 | **1** | **N** |
| ACK type | per packet | **cumulative** | **individual** |
| Timers | 1 | 1 (oldest) | **one per packet** |
| On loss, resend | that packet | **all from the lost one onward** | **only the lost one** |
| Receiver buffering | no | no | yes |
| Max window with k-bit seq | 1 (k = 1) | **2ᵏ − 1** | **2ᵏ⁻¹** |
| Utilisation | 1/(1 + 2a) | min(1, N/(1 + 2a)) | min(1, N/(1 + 2a)) |

**Why the limits (the intuition):** after sending a full window, the sender can't tell whether the receiver **got everything** (and now expects new numbers) or **got nothing** (all ACKs were lost, so old numbers come again). The *new* numbers the receiver expects must never be confusable with *old* numbers that might be retransmitted.
- GBN's receiver expects exactly one number, so the sender just must not reuse **all** 2ᵏ numbers in one window: N ≤ 2ᵏ − 1.
- SR's receiver accepts a whole *window* of numbers, so the old window and the new window must not overlap: 2N ≤ 2ᵏ.

**What TCP uses (preview of note 07):** a hybrid.
- **Cumulative ACKs** like GBN, but the receiver **buffers out-of-order data** like SR.
- Optional **SACK** (Selective ACK) tells the sender exactly which blocks arrived.
- **One retransmission timer** (RFC 6298), plus **fast retransmit** after 3 duplicate ACKs.
- 32-bit **byte** sequence numbers, and a window that varies (flow control + congestion control) instead of a fixed N.

### 2.7 Piggybacking and other details
- **Piggybacking:** when data flows both ways, put the ACK inside the next outgoing data packet instead of sending a separate ACK. It saves packets; TCP always does this (every TCP segment carries an ACK field).
- **Delayed ACK:** the receiver waits a little (TCP: up to ~40–200 ms) hoping to piggyback the ACK or ACK two packets at once. It interacts badly with Nagle's algorithm (note 07).
- **Choosing the timeout:** RTT varies, so real protocols measure it continuously and set timeout ≈ smoothed RTT + 4 × its variation (TCP's RTO, note 07).

### 2.8 Where reliability is deliberately **not** used [HFT]
Retransmission costs **at least one RTT** plus a timeout. For data that is useless when late, it's better **not** to retransmit in-line:
- **Market data feeds** (UDP multicast, note 10): each message carries a **sequence number**. The receiver detects a **gap** (got 1001, then 1003) and either takes the copy from a **redundant A/B feed** (the same data sent twice on separate networks) or asks a **separate recovery service**, while continuing with live data.
- Voice and video: a late audio frame is skipped, not resent.
- Lesson: reliability is a **design choice per application**. TCP forces in-order reliable delivery on everything (head-of-line blocking, note 07), which is exactly what latency-critical systems avoid.

### 2.9 How to read this note: must know / name-only / skip
| Must know | Name-only | Skip |
|---|---|---|
| the four problems + tools; stop-and-wait timeline + U = 1/(1+2a); window N ≥ 1 + 2a; GBN vs SR behaviour, ACKs, timers; window limits 2ᵏ − 1 / 2ᵏ⁻¹ and why; what TCP borrows | alternating-bit name; NAK-based protocols; piggybacking | the rdt 2.0/2.1/2.2/3.0 finite-state machines from Kurose (understand the ideas, don't memorise the FSMs) |

---

## 3. Worked Numericals

**N1. Stop-and-wait utilisation.** 1000-byte frames on a 1 Mbps link, one-way propagation 20 ms. ACKs are negligible.
- T_t = 8000 bits / 10⁶ = **8 ms**. a = T_p / T_t = 20 / 8 = 2.5.
- U = 1 / (1 + 2 × 2.5) = 1/6 = **16.7 %** ⇒ throughput = 0.167 × 1 Mbps ≈ **167 kbps**.

**N2. Window and sequence bits for full utilisation (same link).**
- Need N ≥ 1 + 2a = 1 + 5 = **6 frames**.
- GBN: 2ᵏ − 1 ≥ 6 ⇒ k = **3 bits** (max window 7).
- SR: 2ᵏ⁻¹ ≥ 6 ⇒ 2ᵏ⁻¹ = 8 ⇒ k = **4 bits**.

**N3. Satellite link (large a).** 1 Mbps, 1000-bit frames, one-way propagation 270 ms.
- T_t = 1 ms ⇒ a = 270 ⇒ 1 + 2a = **541**.
- Stop-and-wait: U = 1/541 = **0.18 %**.
- GBN with 3-bit sequence numbers (N = 7): U = 7/541 = **1.3 %**. With 7 bits (N = 127): **23.5 %**.
- 100 % needs N ≥ 541: GBN k = 10 (2¹⁰ − 1 = 1023); SR 2ᵏ⁻¹ ≥ 541 ⇒ k = **11**.

**N4. Retransmissions: GBN vs SR.** 8 packets (0–7), window N = 4. Only the **first** transmission of packet 2 is lost; ACKs are never lost.
- **GBN:** send 0 1 2 3. 2 is lost, so the receiver discards 3. ACKs for 0 and 1 slide the window to [2–5], so the sender sends 4 and 5, which are also discarded. Timeout for 2 ⇒ resend **2 3 4 5**. Then 6 and 7.
  Transmissions = 6 + 4 + 2 = **12**.
- **SR:** 0 1 2 3 4 5 (3, 4, 5 buffered), timeout ⇒ resend only **2**, then 6 and 7. Transmissions = 6 + 1 + 2 = **9**.
- With frequent losses the gap grows; GBN wastes up to N − 1 extra packets per loss.

**N5. Why GBN needs N ≤ 2ᵏ − 1.** Let k = 2 (sequence numbers 0–3) and wrongly use N = 4.
```
Sender sends 0 1 2 3. Receiver gets all four and now expects 0 (of the NEXT round). All 4 ACKs are lost.
Sender times out and resends the OLD packet 0.
Receiver expects 0 → accepts the old copy as NEW data   ✗ silent corruption
```
With N = 3, the sender sends 0 1 2, and the receiver then expects **3**. A resent old 0 is recognised as a duplicate. ✓
SR with k = 2, wrongly N = 3: receiver window [0, 1, 2] → after receiving all, it becomes [3, 0, 1]. A resent old 0 falls inside the new window and is accepted ✗. With N = 2 = 2ᵏ⁻¹, the windows [0, 1] and [2, 3] never overlap ✓.

**N6. Window-limited throughput (what TCP faces).** A protocol allows 64 KB in flight; RTT = 50 ms; the link is 1 Gbps.
- Max throughput = window / RTT = 65,536 × 8 / 0.05 ≈ **10.5 Mbps**, about 1 % of the link.
- To fill 1 Gbps: window ≥ R × RTT = 10⁹ × 0.05 / 8 = **6.25 MB**.

**Try it:** a 10 Mbps link, 500-byte frames, one-way propagation 10 ms. (a) Stop-and-wait utilisation? (b) The minimum window for 100 %, and the sequence-number bits for GBN and for SR? (c) GBN with a window of 7: utilisation?
<details><summary>Answer</summary>T_t = 4000/10⁷ = 0.4 ms, a = 25, 1 + 2a = 51. (a) 1/51 ≈ <b>1.96 %</b>. (b) N ≥ 51 ⇒ GBN k = 6 (63), SR k = 7 (64). (c) 7/51 ≈ <b>13.7 %</b>.</details>

---

## 4. Interview Questions

### Basic
- **Q: Why do reliable protocols need sequence numbers?** To detect duplicates (from retransmissions or lost ACKs) and to reorder. Even stop-and-wait needs 1 bit.
- **Q: What is stop-and-wait and why is it slow?** One packet in flight, so the link idles for a whole RTT per packet: U = 1/(1 + 2a). It's terrible on long or fast links.
- **Q: What is a sliding window?** Up to N unacknowledged packets in flight; the window slides as ACKs return. Full speed needs N ≥ 1 + 2a (≈ the bandwidth-delay product in packets).

### Intermediate
- **Q: Go-Back-N vs Selective Repeat?** §2.6 table. GBN: receiver window 1, cumulative ACK, resends everything after a loss. SR: receiver buffers, per-packet ACKs and timers, resends only lost packets. *Follow-up:* "When is GBN fine?" Low loss, simple or cheap receivers.
- **Q: Why is the GBN window at most 2ᵏ − 1 and the SR window at most 2ᵏ⁻¹?** So new sequence numbers can't be mistaken for retransmitted old ones when all ACKs are lost (N5).
- **Q: What does TCP use, GBN or SR?** A hybrid: cumulative ACKs + receiver buffering + SACK + one timer + fast retransmit; byte-based, variable window (§2.6).
- **Q: Cumulative vs individual ACKs: trade-offs?** Cumulative: one ACK covers many and is robust to lost ACKs, but it can't say what arrived after a gap. Individual/SACK: precise, but more state and bytes.

### Advanced / killer follow-ups
- **Q [AMZN]: Design a reliable, ordered protocol on top of UDP.** Sequence numbers; ACKs (cumulative + selective); a retransmission timer from measured RTT; a sliding window sized to the BDP; receiver buffer + in-order delivery; flow control (the receiver advertises free buffer); congestion control (back off on loss). Connection setup/teardown; duplicate detection. That's essentially TCP or QUIC.
- **Q [HFT]: Why don't market-data feeds use TCP or inline retransmission?** Retransmission costs ≥ 1 RTT and blocks later messages (head-of-line blocking), but late prices are worthless. Instead: UDP multicast + sequence numbers for gap detection + A/B redundant feeds + a separate recovery channel (§2.8).
- **Q: The timeout is set too short. What happens? Too long?** Too short: spurious retransmissions waste bandwidth and (in TCP) wrongly shrink the congestion window. Too long: slow loss recovery and idle links. Hence adaptive RTT estimation.
- **Q: A satellite link is at 0.2 % utilisation with stop-and-wait. What do you change?** Use a pipelined window ≥ 1 + 2a (hundreds of packets ⇒ more sequence bits), prefer SR because losses are costly to resend, and consider forward error correction to avoid retransmissions altogether.
- **Q: How does the receiver's buffer size limit the sender?** In SR/TCP the receiver can only hold its window's worth of out-of-order data, so it advertises that limit (**flow control**). Throughput ≤ receiver window / RTT (N6).

---

## 5. Common Traps
- Forgetting the factor 2 in 1 + 2a: the ACK must travel back.
- Using 2ᵏ as the GBN window (it's 2ᵏ − 1) or 2ᵏ − 1 for SR (it's 2ᵏ⁻¹).
- Saying the GBN receiver buffers out-of-order packets. It discards them.
- Saying SR uses cumulative ACKs. It uses individual ACKs (TCP is the hybrid).
- Forgetting that utilisation is capped at 1 when N > 1 + 2a.
- Ignoring the stated assumptions (ACK size, processing time). Write them explicitly in numericals.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Problems → tools | corrupt → checksum · lost → timer + retransmit · duplicate/reorder → sequence numbers |
| a | T_p / T_t (packets that fit one way) |
| Stop-and-wait | U = 1/(1 + 2a); 1-bit seq (alternating bit) |
| Window | U = min(1, N/(1 + 2a)); full needs N ≥ 1 + 2a (= BDP in packets) |
| GBN | receiver window 1, cumulative ACK, 1 timer, resend window; N ≤ 2ᵏ − 1 |
| SR | receiver buffers, individual ACKs, timer per packet, resend lost only; N ≤ 2ᵏ⁻¹ |
| Why limits | new numbers must not collide with old retransmissions when all ACKs are lost |
| TCP | cumulative ACK + buffering + SACK + 1 timer + fast retransmit; byte seq; variable window |
| Window-limited throughput | ≤ window / RTT |
| Timeout | > RTT; adaptive (SRTT + 4·RTTVAR) |
| HFT | no inline retransmission for market data: seq gap detection + A/B feeds + recovery channel |
