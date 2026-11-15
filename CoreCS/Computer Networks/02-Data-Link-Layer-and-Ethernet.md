# 02. Data Link Layer & Ethernet

> **Asked at:** [AMZN] [DESHAW] switch vs router vs hub, how a switch learns, ARP, MAC vs IP, broadcast vs collision domain · [QCOM] [TI] CRC computation, Wi-Fi CSMA/CA, MTU, error detection in firmware · [HFT] cut-through vs store-and-forward switches, MTU/jumbo frames · [NVIDIA] NIC offloads of CRC/checksums, jumbo frames in clusters.
> **Prerequisites:** [01 Network Basics](01-Network-Basics-and-Layered-Models.md) (layers, frame, encapsulation, delays).

---

## 1. Intuition
The network layer (IP) decides the whole route, e.g. "Chennai → Mumbai → London". The **link layer** handles only **one hop**: getting a parcel from *this* sorting office to the *next* one over one truck route.

For that one hop it must:
1. pack the data into a **frame** whose start and end the receiver can find;
2. write the **next device's** address on it (the MAC address);
3. add a **checksum** so the receiver can detect damage;
4. take turns if several devices share the same wire or radio channel.

---

## 2. Deep Dive

### 2.1 What the link layer does and where it runs
| Job | Meaning | Ethernet does it? |
|---|---|---|
| **Framing** | wrap an IP packet into a frame with a header and trailer, and mark where frames start and end | ✅ |
| **Link addressing** | name the sender and receiver *on this link* with **MAC addresses** | ✅ |
| **Error detection** | detect bit errors (CRC); bad frames are **silently dropped** | ✅ |
| **Medium access** | rules for taking turns on a shared medium | only on old shared Ethernet; ✅ on Wi-Fi |
| Reliable delivery (ACK + retransmit) | resend lost frames | ❌ on Ethernet (left to TCP); ✅ on Wi-Fi |

It runs mostly in the **NIC** (network interface card): framing, CRC generation and checking, and MAC address filtering are done in hardware. The **driver** (kernel) hands packets to the NIC and receives them from it through memory rings using DMA (note 09, [COA 13](../Computer%20Organization%20and%20Architecture/13-IO-Interrupts-DMA.md)).

### 2.2 MAC addresses
A **MAC address** (Media Access Control address) is a **48-bit** hardware address, written as 6 bytes in hex: `3c:22:fb:1a:9e:07`.
- The first 3 bytes are the **OUI** (Organisationally Unique Identifier), which identifies the manufacturer. The last 3 are chosen by the manufacturer, so each NIC is globally unique (modern phones randomise their MAC per Wi-Fi network for privacy).
- `ff:ff:ff:ff:ff:ff` = **broadcast**: every device on the LAN accepts it.
- The lowest bit of the first byte = 1 ⇒ **multicast** (a group of devices, e.g. `01:00:5e:…` for IPv4 multicast, note 10).
- A NIC normally accepts only frames addressed to its own MAC, broadcast, or multicast groups it has joined. **Promiscuous mode** accepts every frame (Wireshark/tcpdump use it).

**Why do we need both MAC and IP addresses?**
| | MAC address | IP address |
|---|---|---|
| Structure | **flat** (no location info) | **hierarchical** (network part + host part, note 04) |
| Assigned by | the manufacturer (fixed to the hardware) | the network you join (changes when you move) |
| Scope | delivery **on one link / LAN** | delivery **across the world** |
| Why | a switch can learn a few thousand local MACs | routers can't hold a table of every device on Earth; they need addresses that **aggregate by location** (one table entry per network) |

A good comparison: the MAC is like a person's name (it stays with them), and the IP is like a postal address (it says where they are now).

### 2.3 The Ethernet frame
```
 ┌──────────┬─────┬────────┬────────┬───────────┬──────────────────────┬─────┐
 │ Preamble │ SFD │ Dst MAC│ Src MAC│ EtherType │ Payload (IP packet)  │ FCS │
 │   7 B    │ 1 B │  6 B   │  6 B   │   2 B     │     46 – 1500 B      │ 4 B │
 └──────────┴─────┴────────┴────────┴───────────┴──────────────────────┴─────┘
  └─ physical-layer sync ─┘└──────────── the "frame": 64 – 1518 B ────────────┘
```
| Field | Purpose |
|---|---|
| Preamble + SFD (Start Frame Delimiter) | alternating 1010… bits so the receiver can lock onto the signal; the SFD marks "frame starts now" |
| **Dst MAC** first | lets the NIC (and a cut-through switch) decide what to do after reading just 6 bytes |
| **EtherType** | which protocol is inside: `0x0800` IPv4, `0x86DD` IPv6, `0x0806` ARP, `0x8100` VLAN tag follows |
| Payload | the IP packet. **46 B minimum** (padded if shorter), **1500 B maximum** = the **MTU** |
| **FCS** | a 32-bit **CRC** over the frame. A frame with a bad FCS is dropped; Ethernet never resends it |

- **MTU (Maximum Transmission Unit)** = the largest payload a link carries: **1500 B** on standard Ethernet. **Jumbo frames** (~9000 B MTU) are used inside data centres and storage networks: fewer frames, less per-packet CPU work. *Software consequence:* a TCP segment's data is at most MTU − 20 (IP) − 20 (TCP) = **1460 B** (the MSS, note 07). An IP packet bigger than the next link's MTU must be fragmented or dropped (note 04).
- **Minimum frame of 64 B** comes from old shared Ethernet's collision detection (§2.6, N4).
- How frame boundaries are found: Ethernet marks the start with the preamble/SFD and the end with an enforced idle gap (12 byte-times). Older serial protocols (HDLC, PPP) use a **flag byte** `01111110` plus **bit stuffing** (after five consecutive 1s, the sender inserts a 0, so the flag can't appear inside the data; N5).

### 2.4 Error detection: parity, Internet checksum, CRC
The sender computes extra **check bits** from the data. The receiver recomputes them and compares. A mismatch means the frame is corrupted, so it's dropped.

| Method | How | Detects | Used in |
|---|---|---|---|
| **Single parity bit** | 1 extra bit so the count of 1s is even | any **odd** number of flipped bits; misses 2 flips | old serial links, memory (with ECC variants) |
| 2-D parity | parity per row and per column | can **locate and correct** 1 flipped bit | teaching example |
| **Internet checksum** | 16-bit one's-complement sum of all 16-bit words, then inverted (N2) | most errors; weak (misses reordered words) | IPv4 header, **TCP, UDP** |
| **CRC** (Cyclic Redundancy Check) | treat the bits as a polynomial, divide by a generator polynomial G, and send the remainder (N1) | **all burst errors ≤ r bits** (r = degree of G), all odd-bit errors (if G has factor x+1), almost everything else | **Ethernet FCS (CRC-32)**, Wi-Fi, disks, ZIP, PNG |

**How CRC works (just the steps):**
1. Pick a generator G of r + 1 bits (degree r). Example: G = 1101 (x³ + x² + 1), so r = 3.
2. Append **r zeros** to the data.
3. Divide by G using **XOR instead of subtraction** (mod-2 arithmetic: no carries or borrows).
4. The **r-bit remainder** is the CRC. Send data + CRC.
5. The receiver divides the whole received frame by G: **remainder 0 ⇒ accept**; otherwise ⇒ error.

CRC is a few XOR gates and shift registers in hardware, so NICs compute it at line rate for free.

**Why have checksums at several layers?** The Ethernet CRC protects only **one hop**. A router can corrupt a packet in its own memory, then compute a *fresh, valid* CRC for the next hop. So TCP/UDP add an **end-to-end** checksum, checked only by the final receiver. This is the **end-to-end argument**: only the endpoints can truly verify the data. Applications that care deeply (storage, downloads) add their own strong hash on top (SHA-256, CRC32C).

**Error correction (name-level):** a code where any two valid codewords differ in at least **d** bits (*minimum Hamming distance* d) can **detect d − 1** errors and **correct ⌊(d − 1)/2⌋**. Example: Hamming(7,4) has d = 3, so it corrects 1 error. It's used in ECC memory and in Wi-Fi/5G radio layers (as FEC, *forward error correction*). Wired Ethernet just detects and drops.

### 2.5 Hubs, switches and the two kinds of "domain"
- **Hub (obsolete):** repeats every incoming bit out of **every** port. All devices share one medium, so only one can transmit at a time; if two do, their signals **collide**.
- **Switch:** each port is a separate link. It forwards a frame **only out of the port where the destination lives**, using a learned table. Every link is **full-duplex** (send and receive at the same time), so there are **no collisions**.
- **Collision domain** = the set of devices whose transmissions can collide. Hub: all ports form one domain. Switch: each port is its own domain (and with full duplex, collisions are impossible).
- **Broadcast domain** = the set of devices that receive a broadcast frame. A switch **forwards broadcasts to all ports**, so the whole switched LAN is **one broadcast domain**. **Routers stop broadcasts** (and so do VLANs, §2.9).

### 2.6 Sharing one medium: CSMA/CD (old Ethernet) and CSMA/CA (Wi-Fi)
When several devices share a medium (a hub, a coaxial cable, a radio channel), they need a **medium access control (MAC) protocol**: the rules for taking turns.

**CSMA/CD** (Carrier Sense Multiple Access with Collision Detection), used by classic shared Ethernet:
1. **Carrier sense:** listen; if someone is transmitting, wait.
2. If the medium is idle, transmit, **while still listening**.
3. **Collision detected** (what's on the wire differs from what you sent) ⇒ stop, send a short **jam** signal so everyone notices.
4. **Binary exponential backoff:** after the n-th collision, wait a random K × 512 bit-times, with K chosen from {0, …, 2ⁿ − 1} (n capped at 10). Give up after 16 attempts. More collisions mean a wider range, which spreads contenders out.
5. **Why a 64-byte minimum frame:** a sender detects a collision only **while it is still transmitting**. The frame must therefore last at least one round trip across the longest allowed cable, so the collision signal gets back before the sender finishes (N4).

**Today, switched full-duplex Ethernet has no collisions, so CSMA/CD is effectively unused.** It still matters because interviews ask about it and it explains the 64-byte minimum.

**CSMA/CA** (Collision **Avoidance**), used by Wi-Fi (802.11):
- A radio **can't detect collisions while sending**: its own signal drowns out everyone else's. There's also the **hidden-terminal problem**: A and C can both reach access point B but can't hear each other, so carrier sense doesn't help.
- So Wi-Fi works like this:
  1. Wait until the channel has been idle for a short gap.
  2. Count down a **random backoff**, then send.
  3. The receiver replies with a **link-layer ACK**. **No ACK ⇒ assume a collision or loss ⇒ back off longer and retransmit.**
  4. Optionally, use **RTS/CTS** (Request-To-Send / Clear-To-Send) first, so hidden terminals hear the CTS and stay quiet.
- *Software consequences:* Wi-Fi is half-duplex and shared, with link-layer retransmissions and rates that change with signal strength. So latency **varies by milliseconds** (jitter) and the occasional loss is *not* congestion, which confuses classic TCP (note 07). [QCOM] [APPLE]

Name-only (GATE-style): **pure ALOHA** max efficiency 1/(2e) ≈ **18.4 %**; **slotted ALOHA** 1/e ≈ **36.8 %**.

### 2.7 How a switch learns: the MAC address table
A switch keeps a table **MAC address → port**, also called the CAM table (content-addressable memory, a hardware lookup table). It fills the table **by itself** by watching traffic:
```
on frame arriving on port P with (src S, dst D):
   1. LEARN:   table[S] = P            (S is reachable via P; entry ages out after ~300 s)
   2. FORWARD:
        D is broadcast/multicast or D not in table → FLOOD out of every port except P
        table[D] == P                              → FILTER (drop: destination is on the same side)
        table[D] == Q                              → FORWARD out of port Q only
```
Nobody configures it; it's **self-learning** and plug-and-play (N3 traces it).
- **Store-and-forward switch:** receives the whole frame, checks the CRC, then forwards. Its delay includes one full transmission time.
- **Cut-through switch:** starts forwarding as soon as it has read the **destination MAC** (first 6 bytes after the preamble). That's ~100–300 ns per hop instead of µs, but a corrupted frame may be forwarded. **[HFT]** exchanges and trading networks use cut-through (often ultra-low-latency switches at ~50–100 ns or less; note 10).
- **Loops:** if switches are cabled in a loop (for redundancy), a broadcast frame circulates forever and multiplies (a *broadcast storm*), because Ethernet frames have no TTL. The **Spanning Tree Protocol (STP)** fixes this by disabling some links so the active topology is a tree. Name-level only.
- **MAC flooding attack:** fill the table with fake source MACs. The switch then floods all traffic like a hub, and an attacker can sniff it. Defence: port security (limit MACs per port).

### 2.8 ARP: finding the MAC address for an IP address
The kernel knows the destination **IP** (from the socket), but the Ethernet frame needs a destination **MAC**. **ARP** (Address Resolution Protocol) asks "who has this IP?" on the LAN.

**Case 1: destination on the same LAN** (same subnet, note 04). A = 192.168.1.10 sends to B = 192.168.1.20:
```
1. A checks its ARP cache (IP → MAC table, `ip neigh`). Miss.
2. A broadcasts:  dst MAC ff:ff:ff:ff:ff:ff   "Who has 192.168.1.20? Tell 192.168.1.10 (MAC aa:..)"
3. Every host receives it; only B answers, by unicast to A:  "192.168.1.20 is at bb:.."
4. A caches (192.168.1.20 → bb:..) for minutes, then sends the IP packet in a frame to bb:..
   (B also caches A's mapping from the request, so its reply needs no ARP.)
```
**Case 2: destination on another network** (e.g. 8.8.8.8). A does **not** ARP for 8.8.8.8, since that host isn't on its link. It looks up its routing table, finds the **default gateway** (its router, e.g. 192.168.1.1), and ARPs for the **router's MAC**. The frame goes to the router's MAC, but the IP packet inside still says destination 8.8.8.8. This is §2.5 of note 01 in action: **MACs per hop, IP end to end**.

- **Gratuitous ARP:** a host announces its own IP → MAC unprompted (after boot, or when a backup server takes over an IP during failover).
- **ARP spoofing:** ARP has no authentication. An attacker replies "192.168.1.1 is at *my* MAC", so victims send gateway traffic to the attacker (a man-in-the-middle). Defences: static ARP entries, Dynamic ARP Inspection on switches, and encryption (TLS) so stolen traffic is useless.
- IPv6 replaces ARP with **NDP** (Neighbour Discovery Protocol), which uses ICMPv6 multicast instead of broadcast.

### 2.9 VLANs: several logical LANs on one physical switch
A **VLAN** (Virtual LAN) splits one switch into separate broadcast domains: ports in VLAN 10 (say, Engineering) never see VLAN 20's (Finance) broadcasts or frames. Traffic between VLANs must go through a router. Benefits: isolation, security, smaller broadcast domains.
- Between switches, frames carry a 4-byte **802.1Q tag** after the source MAC, containing a **12-bit VLAN ID** (4096 values; 0 and 4095 are reserved ⇒ **4094 usable**) and a 3-bit priority field (QoS).
- Clouds needed more than 4094 isolated networks, so they use **VXLAN** (a 24-bit ID ⇒ ~16 M networks, Ethernet frames tunnelled inside UDP). Name-level, but useful for AWS VPC discussions.

### 2.10 How to read this note: must know / name-only / skip
| Must know | Name-only | Skip |
|---|---|---|
| link-layer jobs; MAC vs IP and why both; Ethernet frame fields, MTU 1500, 64 B min; CRC steps + one computation; Internet checksum; end-to-end argument; hub vs switch vs router; collision vs broadcast domain; switch learning; cut-through vs store-and-forward; ARP both cases; CSMA/CD steps; why Wi-Fi uses CSMA/CA; VLAN purpose | ALOHA efficiencies; STP; Hamming codes; VXLAN; bit stuffing | PHY encodings (8b/10b, 64b/66b, PAM4), Ethernet auto-negotiation, Token Ring |

---

## 3. Worked Numericals

**N1. CRC.** Data = `101101`, generator G = `1101` (x³ + x² + 1, degree r = 3). Find the transmitted frame.
1. Append 3 zeros: `101101 000`.
2. Divide using XOR. At each step, if the leading bit is 1, XOR with 1101; if it's 0, XOR with 0000. Then drop the leading bit and bring down the next one:
```
1011 ⊕ 1101 = 0110 → keep 110, bring down 0 → 1100
1100 ⊕ 1101 = 0001 → keep 001, bring down 1 → 0011
0011 (lead 0)       → keep 011, bring down 0 → 0110
0110 (lead 0)       → keep 110, bring down 0 → 1100
1100 ⊕ 1101 = 0001 → keep 001, bring down 0 → 0010
0010 (lead 0)       → remainder = 010
```
3. CRC = `010` ⇒ transmit **`101101010`**.
4. Check at the receiver: dividing `101101010` by `1101` gives remainder `000` ⇒ accept. If the channel flips the last bit (`101101011`), the remainder is non-zero ⇒ drop.

**N2. Internet checksum.** Three 16-bit words: 0xF000, 0x2001, 0x0100.
- 0xF000 + 0x2001 = 0x1_1001. There's a carry out of 16 bits, so **wrap it around** (add it back at the bottom): 0x1001 + 1 = 0x1002.
- 0x1002 + 0x0100 = 0x1102.
- Checksum = one's complement (flip all bits) = **0xEEFD**.
- Receiver: adds all words **plus** the checksum: 0x1102 + 0xEEFD = 0xFFFF ⇒ all ones ⇒ OK.

**N3. Switch learning trace.** A 4-port switch with an empty table. A is on port 1, B on 2, C on 3, D on 4.
| Frame | Learn | Table after | Action |
|---|---|---|---|
| A → B | A@1 | {A:1} | B unknown ⇒ **flood** to 2, 3, 4 |
| B → A | B@2 | {A:1, B:2} | A known ⇒ forward **only to 1** |
| C → A | C@3 | {A:1, B:2, C:3} | forward to 1 |
| A → D | (A@1 again) | same | D unknown ⇒ **flood** to 2, 3, 4 |
| D → broadcast (ARP) | D@4 | {A:1, B:2, C:3, D:4} | broadcast ⇒ flood to 1, 2, 3 |
| A → C | — | same | forward to 3 only |

**N4. Minimum frame size for CSMA/CD.** Shared 10 Mbps Ethernet, longest cable 2500 m, signal speed 2 × 10⁸ m/s, ignoring repeater delays.
- Worst-case round trip = 2 × 2500 / (2 × 10⁸) = 25 µs.
- The frame must still be transmitting after 25 µs ⇒ L_min = R × 2d/s = 10⁷ × 25 × 10⁻⁶ = **250 bits**.
- The standard chose **512 bits = 64 bytes** to leave margin for repeaters. *Consequence:* at 1 Gbps, the same 512 bits would allow only ~1/100 of the cable length, which is one reason gigabit and faster Ethernet are switched and full-duplex only.

**N5. Bit stuffing.** Data `0111111111100` (ten 1s in a row), flag `01111110`.
- After every five consecutive 1s, insert a 0: `0 11111 0 11111 0 00` ⇒ **`011111011111000`** (13 → 15 bits).
- The receiver deletes any 0 that follows five 1s. Six 1s in a row can then appear only in the flag.

**N6. [HFT] Store-and-forward vs cut-through.** A 1518-byte frame at 10 Gbps through 3 switches.
- Store-and-forward: each switch waits for the full frame: 1518 × 8 / 10¹⁰ = 1.21 µs per hop ⇒ ~3.6 µs of extra delay (plus processing).
- Cut-through (~0.3 µs per hop after reading the header): ≈ **0.9 µs**. For a 64-byte frame the gap is smaller (51 ns per hop to serialize), but cut-through still wins.

**Try it:** (a) Data `11010011101100`, G = `1011` (x³ + x + 1). Find the CRC. (b) Checksum of the words 0x4500 and 0x0030.
<details><summary>Answer</summary>(a) Append 000 and divide by 1011 ⇒ remainder <b>100</b>; transmit 11010011101100 100. (b) 0x4500 + 0x0030 = 0x4530 ⇒ checksum = ~0x4530 = <b>0xBACF</b> (check: 0x4530 + 0xBACF = 0xFFFF).</details>

---

## 4. Interview Questions

### Basic
- **Q: What does the data link layer do?** Framing, MAC addressing for one hop, error detection (CRC; bad frames dropped), and medium access on shared media. Ethernet doesn't retransmit; Wi-Fi does (link-layer ACKs).
- **Q: Hub vs switch vs router?** Hub: L1, repeats bits to all ports (one collision domain). Switch: L2, forwards by MAC using a learned table, one broadcast domain. Router: L3, forwards by IP between networks and stops broadcasts.
- **Q: What is a MAC address? Why do we need IP addresses too?** 48-bit flat hardware ID, good for one LAN. IP is hierarchical, so routers can aggregate by location; a flat global table of every MAC is impossible (§2.2).
- **Q: What is ARP?** It maps an IP to a MAC on the local link: broadcast request, unicast reply, cached. For remote destinations, ARP for the default gateway's MAC.

### Intermediate
- **Q: How does a switch build its table? What if the destination is unknown?** It learns source MAC → port from every frame; unknown or broadcast destinations are flooded; entries age out (§2.7, N3). *Follow-up:* "What if the table fills up?" New MACs get flooded like a hub (the basis of the MAC-flooding attack); defend with port security.
- **Q: Collision domain vs broadcast domain?** Collision: devices that can collide (each switch port is its own; full duplex has none). Broadcast: devices that get a broadcast (the whole switched LAN or VLAN; routers bound it).
- **Q: Explain CSMA/CD. Why is there a minimum frame size?** Listen, transmit, detect collision, jam, binary exponential backoff. The frame must outlast a round trip so the sender is still transmitting when the collision comes back (N4). Irrelevant on full-duplex switched Ethernet.
- **Q [QCOM]: Why does Wi-Fi use CSMA/CA instead of CSMA/CD?** A radio can't hear others while transmitting, and hidden terminals can't hear each other. So: random backoff, a link-layer ACK for every frame, optional RTS/CTS.
- **Q: CRC vs checksum? Why have checksums at several layers?** CRC is stronger (catches all bursts ≤ r bits) and cheap in hardware, so it's used at the link layer. The Internet checksum is a weak but simple software sum for IP/TCP/UDP. The link CRC covers one hop only; the end-to-end checksum catches corruption inside routers (end-to-end argument).
- **Q: What is the MTU and why does it matter to software?** The largest link payload (1500 B on Ethernet) ⇒ TCP MSS 1460. Larger IP packets fragment or drop (note 04). Jumbo frames (9000) cut per-packet overhead in data centres.

### Advanced / killer follow-ups
- **Q [HFT]: Cut-through vs store-and-forward. Which do you want and what's the catch?** Cut-through forwards after reading the destination MAC (~100s of ns vs µs). The catch: corrupted frames get forwarded (the CRC is checked later), and it only helps when input and output ports run at the same speed.
- **Q: Two hosts on the same switch can't talk, but each can reach the router. What do you check?** That they're in the same VLAN/subnet, the ARP cache (`ip neigh`), the MAC table on the switch, and duplicate IPs (shows up as ARP flapping).
- **Q: What is ARP spoofing and how do you defend against it?** Forged ARP replies redirect traffic to the attacker (man-in-the-middle). Defences: Dynamic ARP Inspection, static entries for gateways, and end-to-end encryption.
- **Q: Why does Ethernet have no TTL, and what problem does that cause?** It was designed for a single segment. With switch loops, broadcasts circulate forever (broadcast storm), which STP prevents by blocking redundant links.
- **Q [NVIDIA]: Who computes CRCs and checksums on a 100 Gbps NIC?** The NIC hardware: Ethernet CRC always, and TCP/UDP/IP checksums through **checksum offload** (note 09). Software checksumming at 100 Gbps would eat several cores.
- **Q: How many VLANs can 802.1Q support, and what do clouds do beyond that?** 12-bit ID ⇒ 4094 usable. Clouds use VXLAN (24-bit ⇒ 16 M), which tunnels Ethernet frames inside UDP.

---

## 5. Common Traps
- Saying a switch reduces the *broadcast* domain. It splits collision domains; routers and VLANs split broadcast domains.
- ARPing for a remote IP. You ARP for the **gateway's** MAC.
- Thinking Ethernet retransmits corrupted frames. It drops them; TCP (or the application) recovers.
- Forgetting to append r zeros before CRC division, or using ordinary subtraction instead of XOR.
- Forgetting the end-around carry in the Internet checksum.
- Claiming CSMA/CD is used on modern Ethernet (full-duplex switched links have no collisions).
- Mixing up frame size (64–1518 B) with MTU (payload ≤ 1500 B).

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Link-layer jobs | framing, MAC addressing (1 hop), CRC error detection (drop), medium access |
| MAC | 48-bit, OUI + NIC ID; ff:ff… broadcast; flat ⇒ LAN only; IP hierarchical ⇒ routable |
| Ethernet frame | preamble 7 + SFD 1 · dst 6 · src 6 · type 2 · payload 46–1500 · FCS 4 ⇒ 64–1518 B |
| EtherType | 0x0800 IPv4 · 0x86DD IPv6 · 0x0806 ARP · 0x8100 VLAN |
| MTU | 1500 ⇒ MSS 1460; jumbo 9000 |
| CRC | append r zeros, XOR-divide by G, remainder = CRC; catches bursts ≤ r |
| Checksum | 16-bit one's-complement sum + end-around carry, inverted; IP/TCP/UDP |
| End-to-end | link CRC per hop; TCP/UDP checksum end to end |
| Hub / switch / router | L1 repeat / L2 learn + forward by MAC / L3 by IP, stops broadcasts |
| Switch algorithm | learn src → port; unknown/broadcast ⇒ flood; same port ⇒ filter; else forward |
| Cut-through | forward after dst MAC (~100s ns) vs store-and-forward (+ L/R per hop) |
| ARP | broadcast "who has IP?", unicast reply, cache; remote ⇒ gateway MAC |
| CSMA/CD | sense, send, detect, jam, exponential backoff; 64 B min frame; obsolete with switches |
| CSMA/CA (Wi-Fi) | backoff + link ACK + RTS/CTS; hidden terminal; jitter, loss ≠ congestion |
| VLAN | separate broadcast domains; 802.1Q tag, 12-bit ID (4094); VXLAN 24-bit |
