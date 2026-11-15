# 01. Network Basics & Layered Models

> **Asked at:** [AMZN] [DESHAW] OSI vs TCP/IP layers, what each layer does, encapsulation, packet vs circuit switching · [HFT] transmission vs propagation delay, why microwave beats fibre, serialization delay of a small order message · [NVIDIA] bandwidth-delay product, why bandwidth ≠ latency · [QCOM] [TI] which layers live in hardware, firmware, kernel and app.
> **Prerequisites:** none. This is the foundation for every later CN note.

---

## 1. Intuition
Sending data over the Internet is like sending a letter through the post:
- You write a **letter** (your application's data).
- You put it in an envelope with an **apartment number** (the *port*: which program on the computer should get it).
- That goes inside a bigger envelope with the **street address** (the *IP address*: which computer, anywhere in the world).
- The post office moves it **one sorting office at a time**. Each truck only knows how to reach the *next* office (the *link layer*: one hop at a time).

Each step adds its own envelope (**header**) and only reads its own envelope. That separation into **layers** is the core idea of this note.

---

## 2. Deep Dive

### 2.1 Basic vocabulary (every later note uses these)
| Term | Meaning |
|---|---|
| **Host / end system** | a computer at the edge that runs applications: laptop, phone, server |
| **Link** | a physical connection between two neighbouring devices: copper cable, fibre, Wi-Fi radio |
| **Packet** | a small chunk of data (typically ≤ 1500 bytes) with a **header** in front saying where it goes. Big data is cut into many packets |
| **Header** | extra bytes added in front of the data by a protocol, holding addresses, lengths, sequence numbers and similar fields |
| **Protocol** | the rules two sides agree on: the **message format**, the **order** of messages, and the **action** to take on each message (e.g. "on SYN, reply SYN-ACK") |
| **Switch** | a device that forwards packets inside one local network (LAN) using hardware (MAC) addresses (note 02) |
| **Router** | a device that forwards packets **between networks** using IP addresses (notes 04–05) |
| **LAN / WAN** | Local Area Network (one building or office) / Wide Area Network (across cities and countries) |
| **ISP** | Internet Service Provider. The Internet is a "network of networks": thousands of ISPs and companies connected together |
| **Bandwidth (link rate) R** | how many **bits per second** a link can push onto the wire, e.g. 1 Gbps = 10⁹ bits/s |
| **Latency** | how long one bit (or one message) takes to get from sender to receiver, in seconds |
| **Throughput** | the rate at which useful data **actually** arrives end to end (≤ the slowest link) |
| **RTT** | round-trip time: the time to send a small message and get the reply back |

> **Units trap:** in networking, **k / M / G are powers of 10** (1 Mbps = 10⁶ bits/s), and rates are in **bits** (lower-case b). File sizes are usually in **bytes** (upper-case B, often powers of 2). 1 byte = 8 bits. Many numerical mistakes come from mixing these up.

### 2.2 Circuit switching vs packet switching
- **Circuit switching** (the old telephone network): before talking, reserve a fixed path and a fixed share of every link on it, e.g. one time slot out of 24. The reservation stays even when nobody speaks.
- **Packet switching** (the Internet): no reservation. Each packet is sent on its own; links are shared on demand. When too many packets arrive at once, they wait in a **queue** (buffer) at the router; if the buffer is full, the packet is **dropped** (packet loss).

| | Circuit switching | Packet switching |
|---|---|---|
| Setup before sending | yes (reserve the path) | no |
| Guaranteed rate / delay | ✅ | ❌ (queues ⇒ variable delay, loss) |
| Link use when a user is idle | wasted | used by others (**statistical multiplexing**) |
| Good for | constant-rate voice | bursty data (web, files), which is most traffic |

**Statistical multiplexing** means sharing a link among many users on the bet that they won't all send at once. Example: a 10 Mbps link, users who need 1 Mbps while active but are active only 10 % of the time. Circuit switching can admit only 10 users. Packet switching can admit ~35, and the chance that more than 10 are active at once is tiny (≈ 0.04 %).

**Store-and-forward:** a router must receive the **whole packet** before it starts sending it on the next link, because it has to read the header and check the packet for errors. So every hop adds one full packet's transmission time (N2).

### 2.3 Why layers?
A network has to solve many separate problems: sending bits on a wire, reaching the next device, finding a path across the world, recovering lost data, and serving the application. **Layering** gives each problem to one layer. Each layer:
- **uses** the service of the layer below, without knowing how it works;
- **offers** a service to the layer above;
- talks to the **same layer on the other machine** through its own header.

That gives **modularity**: Wi-Fi can replace Ethernet without changing TCP, and HTTP runs unchanged over any network.

### 2.4 The two models: OSI (7 layers) and TCP/IP (the one actually used)
| OSI layer | TCP/IP layer | Job (one line) | Unit of data | Address used | Examples | Lives in |
|---|---|---|---|---|---|---|
| 7 Application | **Application** | the app's own protocol | message | names / URLs | HTTP, DNS, SMTP, SSH, FIX (trading) | user-space program |
| 6 Presentation | (part of the app) | data format, encryption, compression | | | TLS, JSON, protobuf | user-space libraries |
| 5 Session | (part of the app) | open/close/resume conversations | | | TLS sessions, RPC | user-space libraries |
| 4 Transport | **Transport** | **process-to-process** delivery; optional reliability and ordering | **segment** (TCP) / datagram (UDP) | **port** (16-bit) | TCP, UDP, QUIC (over UDP) | OS kernel |
| 3 Network | **Network (Internet)** | **host-to-host** delivery across many networks; routing | **packet** (datagram) | **IP address** | IPv4, IPv6, ICMP | OS kernel (+ routers) |
| 2 Data link | **Link** | deliver a frame to the **next device** on the same link | **frame** | **MAC address** (48-bit) | Ethernet, Wi-Fi (802.11) | NIC hardware + driver |
| 1 Physical | (Physical) | turn bits into signals on the medium | bits | — | copper, fibre, radio | NIC / PHY hardware |

- **OSI** (Open Systems Interconnection) is a 7-layer *reference model* used for teaching and vocabulary ("an L7 load balancer", "an L2 switch"). **TCP/IP** is what the Internet actually runs: 4 layers, or 5 if Physical is counted separately. OSI's Presentation and Session layers are folded into the application.
- **Mnemonic** (7 → 1): **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing.
- **Where each layer runs:** application, presentation and session in your program and its libraries; transport and network **in the kernel** (that's why `send()` is a system call); link in the **NIC** (network card) hardware plus its driver; physical in the NIC's PHY chip and the cable.
- **Which devices look at which layers:**

| Device | Reads up to | Why |
|---|---|---|
| Hub / repeater (obsolete) | L1 | just copies bits to every port |
| **Switch** | **L2** | forwards by destination MAC address |
| **Router** | **L3** | forwards by destination IP address |
| Firewall / NAT / L4 load balancer | L4 | looks at ports and connection state |
| L7 load balancer / proxy / CDN | L7 | looks at the HTTP URL, headers, cookies |
| End host | all layers | it is where the data starts and ends |

### 2.5 Encapsulation: how the envelopes nest
Going **down** the stack on the sender, each layer adds its header in front of what it got from above. Going **up** on the receiver, each layer removes its own header and passes the rest up.
```
Application   [                 data (e.g. HTTP request)                  ]
Transport     [TCP hdr 20 B][               data                          ]   = segment
Network       [IP hdr 20 B][TCP hdr][               data                  ]   = packet
Link          [Eth hdr 14 B][IP hdr][TCP hdr][         data       ][FCS 4 B]  = frame
Physical      preamble 8 B + frame + inter-frame gap 12 B  →  bits on the wire
```
- **FCS** (Frame Check Sequence) is a 4-byte CRC error-detection code at the **end** of an Ethernet frame (note 02). The **preamble** is a fixed bit pattern that lets the receiver lock onto the signal. The **inter-frame gap** is required idle time between frames.
- Typical header sizes: Ethernet 14 B + 4 B FCS, IPv4 20 B (IPv6 40 B), TCP 20 B (up to 60 B with options), UDP 8 B.

**What happens at each hop:** a packet from your laptop to a server crosses several switches and routers.
```
Laptop ──► Switch ──► Router R1 ──► ... ──► Router Rn ──► Server
 L1–L7      L1–L2      L1–L3                 L1–L3        L1–L7
```
- A **switch** reads the Ethernet header and forwards the frame unchanged.
- A **router** removes the old Ethernet header, reads the IP header, decrements the **TTL** (time-to-live: a hop counter that stops packets from looping forever, note 04), updates the IP checksum, and wraps the packet in a **new** link header for the next link.
- **IP addresses stay the same end to end; MAC addresses change at every router hop.** (The exception is NAT, note 04.)

> **"A router is a layer-3 device, so where are its L1 and L2?"** In **every interface**. "L3 device" means L3 is the *highest layer it uses to forward*, not that L1/L2 are missing.
> - **L1:** each interface has a PHY chip or optical transceiver (e.g. an SFP module) that converts bits ⇄ signals.
> - **L2:** each interface has a MAC controller with **its own MAC address** that builds/parses frames and checks the CRC.
> - **L3:** shared by all interfaces. The forwarding engine (ASIC or CPU) does the LPM lookup and TTL − 1, and picks the output interface.
> ```
> Laptop                  Router R1                            Router R2                      Server
> App / TCP                                                                                   App / TCP
> IP ──────────────────── IP (LPM, TTL−1) ──────────────────── IP (LPM, TTL−1) ───────────── IP
> Wi-Fi MAC ── link 1 ─── Wi-Fi MAC │ Eth MAC ── link 2 ─────── Eth MAC │ Eth MAC ── link 3 ── Eth MAC
> Wi-Fi PHY ~~ radio ~~~~ Wi-Fi PHY │ Eth PHY ══ fibre ════════ Eth PHY │ Eth PHY ══ copper ══ Eth PHY
>                        (iface a)   (iface b)                 (iface x)   (iface y)
> ```
> - **The link layer is per link.** The two ends of one link pair up (laptop NIC ↔ R1 interface a). A **frame** (Ethernet header + FCS around the IP packet) exists only while it crosses **one link**: the device **sending** onto that link builds it; the device **receiving** it checks it and **throws the wrapper away**. It's never carried further:
>   ```
>   Laptop builds frame 1:   [Eth dst=R1-a src=laptop][IP ... TTL=64 | data][FCS]   ── link 1 ──►
>   R1 iface a: FCS ✓, dst MAC is mine ✓ → REMOVES Eth hdr + FCS → frame 1 no longer exists
>   R1 keeps only the packet: [IP ... TTL=64 | data] → lookup → TTL 63 → out of iface b
>   R1 iface b builds a NEW frame 2: [Eth dst=R2-x src=R1-b][IP ... TTL=63 | data][new FCS] ── link 2 ──►
>   ```
>   Analogy: the IP packet is a sealed letter; the frame is the courier bag for **one leg**. Each sorting office (router) opens the bag, throws it away, reads the letter's address, and puts the letter in a **new bag** for the next leg. The old frame can't be reused: its MACs mean "laptop → R1", the next link may be a different technology (a different header format), and the FCS must be recomputed (TTL changed). **A switch forwards the same frame; a router forwards the packet inside a new frame.**
> - **Each hop can use a different link type** (Wi-Fi, fibre Ethernet, copper, 5G, DSL). IP is the common layer above all of them, which is the point of layering (§2.3).
> - **Inside R1:** iface a PHY decodes → MAC checks CRC + "is the dst MAC mine?" → strip frame → IP lookup, TTL − 1 → choose iface b → ARP for the next hop's MAC → iface b builds a **new** frame (src = b's MAC) → PHY transmits.
>
> | Link | Src MAC | Dst MAC | Src IP | Dst IP | TTL |
> |---|---|---|---|---|---|
> | 1 (Wi-Fi) | laptop | R1 iface a | laptop | server | 64 |
> | 2 (fibre) | R1 iface b | R2 iface x | laptop | server | 63 |
> | 3 (copper) | R2 iface y | server | laptop | server | 62 |
> - Routers also run **higher layers for their own traffic** (BGP over TCP, SSH management: the control plane). Forwarding *other people's* packets (the data plane) stops at L3.

### 2.6 The four delays a packet suffers at each hop
```
          ┌──────────── one hop (router A → router B) ─────────────┐
 packet → [processing] → [queuing] → [transmission] → [propagation] → arrives at B
           check header    wait for     push all bits    bits travel
           pick output     other        onto the link    down the wire
           port            packets
```
| Delay | What it is | Formula / typical value |
|---|---|---|
| **Processing** d_proc | read the header, check for errors, look up the output port | ns to µs (hardware routers ~ns) |
| **Queuing** d_queue | wait behind other packets in the output buffer | 0 to ms; depends on load (see below) |
| **Transmission** (serialization) d_trans | time to **push all the packet's bits onto the link** | **L / R** (L = packet size in bits, R = link rate) |
| **Propagation** d_prop | time for **one bit to travel** the length of the link | **d / s** (d = distance, s ≈ 2 × 10⁸ m/s in fibre or copper ≈ **5 µs per km**) |

Nodal delay = d_proc + d_queue + d_trans + d_prop.

**Transmission vs propagation (the classic confusion):** transmission depends on **packet size and link speed**, not distance. Propagation depends on **distance and the medium**, not packet size or link speed. A highway analogy:
- *transmission* = the time for a 10-car convoy to pass through the toll booth, one car at a time;
- *propagation* = the time for one car to drive to the next town.

Making the link faster (higher R) shrinks transmission only. Nothing shrinks propagation except a shorter path or a faster medium: light in air or vacuum is ~1.5× faster than in fibre, which is why HFT firms use **microwave** links (note 10).

**Queuing delay and traffic intensity:** let a = packets arriving per second and L = bits per packet. **Traffic intensity = L·a / R**, the arrival rate in bits/s divided by the link rate.
- ≪ 1: queues are almost always empty, so queuing delay ≈ 0.
- → 1: queuing delay grows very fast (packets arrive in bursts).
- \> 1: the queue grows without limit ⇒ the buffer overflows ⇒ **packet loss**. Loss is how the Internet signals congestion (note 07).

### 2.7 Throughput, RTT and the bandwidth-delay product
- **End-to-end throughput = the rate of the slowest (bottleneck) link** on the path. A 10 Gbps server behind a 100 Mbps home connection gets ≤ 100 Mbps.
- **RTT** ≈ 2 × (sum of one-way delays). Measure it with `ping host`.
- **Bandwidth-delay product (BDP) = R × RTT** = the number of bits "in flight" when a sender keeps the link full and waits one RTT for acknowledgements. Think of it as the **volume of the pipe**. If a protocol may only have W bits unacknowledged at once and W < BDP, the link sits idle part of the time. This is why TCP windows must be large on fast, long links (notes 03 and 07).

**Latency vs bandwidth:** bandwidth is how *wide* the pipe is; latency is how *long* it is. A small message (an order, an RPC, a DNS query) cares mostly about **latency**. A big transfer (a backup, a video) cares mostly about **bandwidth**. Upgrading 1 → 10 Gbps barely helps a 100-byte request across a continent (N1, N6).

### 2.8 Try these commands (seeing it makes it stick)
| Command | Shows |
|---|---|
| `ping google.com` | RTT to a host (uses ICMP, note 04) |
| `traceroute google.com` (Windows: `tracert`) | every router hop and its RTT (how it works: note 04) |
| `ip addr` / `ipconfig` | your IP and MAC addresses |
| `ss -tn` / `netstat -an` | open TCP connections (note 06) |
| Wireshark / `tcpdump -i any port 443` | real packets with every layer's header decoded |

### 2.9 How to read this note: must know / name-only / skip
| Must know | Name-only | Skip (beyond SW interviews) |
|---|---|---|
| layers + their jobs + unit names + addresses; which devices read which layer; encapsulation; MAC changes per hop and IP doesn't; four delays and formulas; BDP; bottleneck throughput | OSI presentation/session details; statistical-multiplexing probability maths | signal encoding (NRZ, Manchester), modulation, Shannon/Nyquist capacity (only for core EE roles) |

---

## 3. Worked Numericals

**N1. Transmission vs propagation.** A 1500-byte packet on a 100 Mbps link that is 2000 km long. Signal speed 2 × 10⁸ m/s.
- L = 1500 × 8 = 12,000 bits.
- d_trans = L / R = 12,000 / 10⁸ = **120 µs**.
- d_prop = d / s = 2 × 10⁶ m / 2 × 10⁸ m/s = **10 ms**.
- Propagation is ~83× larger. Upgrading to 1 Gbps cuts the total only from 10.12 ms to 10.012 ms.

**N2. Store-and-forward, and why data is cut into packets.** Send a 1 MB (= 8 × 10⁶ bits) message from A to B through 2 routers (3 links), each 10 Mbps. Ignore propagation, processing and queuing.
- **As one big message:** each link needs 8 × 10⁶ / 10⁷ = 0.8 s, and each router must receive it all before forwarding ⇒ 3 × 0.8 = **2.4 s**.
- **As 1000 packets of 8000 bits:** one packet takes 8000 / 10⁷ = 0.8 ms per link. Links work in parallel like a pipeline: the first packet arrives after 3 × 0.8 ms, then one more arrives every 0.8 ms ⇒ (1000 + 3 − 1) × 0.8 ms = **0.8016 s**.
- General formula: N packets over k store-and-forward links ⇒ (N + k − 1) × L/R. Packetising gives ~3× faster delivery here.

**N3. Header overhead and efficiency.** An app sends 100 bytes of data per message over TCP/IPv4/Ethernet.
- Headers: TCP 20 + IP 20 + Ethernet 14 + FCS 4 = 58 B ⇒ frame = 158 B. On the wire add preamble 8 + gap 12 ⇒ **178 B**.
- Efficiency = 100 / 178 = **56 %**. With 1460-byte payloads (the usual maximum): 1460 / (1460 + 58 + 20) = 1460 / 1538 = **94.9 %**.
- Lesson: tiny messages waste bandwidth on headers, which is one reason for batching (and for Nagle's algorithm, note 07).

**N4. Bandwidth-delay product.** A 10 Gbps link from Mumbai to the US East Coast, RTT = 200 ms.
- BDP = 10¹⁰ bits/s × 0.2 s = 2 × 10⁹ bits = **250 MB**.
- If the sender may have only 64 KB unacknowledged (classic TCP without window scaling), throughput ≤ window / RTT = 65,536 × 8 bits / 0.2 s ≈ **2.6 Mbps**, which is **0.026 %** of the link. This is why TCP window scaling exists (note 07).

**N5. Bottleneck throughput.** Server link 1 Gbps → ISP backbone 10 Gbps → home connection 50 Mbps. Download a 500 MB file.
- Throughput = min(1000, 10,000, 50) Mbps = 50 Mbps.
- Time = 500 × 8 × 10⁶ bits / 5 × 10⁷ bits/s = **80 s** (assuming MB = 10⁶ B; with 2²⁰ it's ~84 s).

**N6. [HFT] Serialization delay of a small order.** An 84-byte frame (the minimum Ethernet frame on the wire: 64 B frame + 8 preamble + 12 gap) carries a tiny order.
- At 1 Gbps: 84 × 8 / 10⁹ = **672 ns** per hop. At 10 Gbps: **67.2 ns**. At 25 Gbps: **26.9 ns**.
- Over 4 store-and-forward hops at 10 Gbps: ~270 ns of serialization alone. In HFT, where total budgets are a few µs, higher link rates and **cut-through switches** (which start forwarding before the whole frame arrives, note 10) do matter, even for tiny messages.

**N7. Queuing.** Packets of 1500 B arrive at a 100 Mbps router port at 8000 packets/s on average.
- Traffic intensity = L·a / R = 12,000 × 8000 / 10⁸ = **0.96**. That's close to 1, so expect long and highly variable queues. At 9000 packets/s it's 1.08 > 1 ⇒ the queue grows until packets are dropped.

**Try it:** a 4000-byte message goes over 2 links (1 router in between). Link 1: 8 Mbps, 400 km. Link 2: 4 Mbps, 1000 km. s = 2 × 10⁸ m/s; store-and-forward; no queuing or processing. What is the end-to-end delay?
<details><summary>Answer</summary>L = 32,000 bits. Link 1: trans 32,000/8×10⁶ = 4 ms, prop 400 km × 5 µs/km = 2 ms. Link 2: trans 32,000/4×10⁶ = 8 ms, prop 1000 × 5 µs = 5 ms. Total = 4 + 2 + 8 + 5 = <b>19 ms</b>.</details>

---

## 4. Interview Questions

### Basic
- **Q: Name the OSI layers and what each does.** §2.4 table. Mention the unit at each layer (bits, frame, packet, segment, message) and the address used (MAC, IP, port). *Follow-up:* "Which ones does the Internet actually use?" TCP/IP's 4–5 layers; presentation and session are part of the application.
- **Q: What is a protocol?** An agreement on message format, message order, and the action on each message. Example: TCP's handshake (SYN → SYN-ACK → ACK).
- **Q: Packet switching vs circuit switching?** §2.2. Packet switching shares links on demand (efficient for bursty traffic) but gives variable delay and loss. Circuit switching reserves capacity (guaranteed, but wasted when idle).
- **Q: Bandwidth vs latency vs throughput?** Bandwidth = bits/s a link can carry; latency = time for data to arrive; throughput = actual delivered rate (≤ the bottleneck link).

### Intermediate
- **Q: Transmission delay vs propagation delay?** L/R vs d/s. Transmission depends on packet size and link rate; propagation depends on distance and medium. Only a shorter path or faster medium reduces propagation.
- **Q: What does a router change in a packet? What does a switch change?** A switch forwards the frame unchanged (by MAC). A router strips the link header, decrements TTL, recomputes the IP header checksum, and adds a new link header with new MACs. IPs are unchanged unless NAT is involved.
- **Q: A router is an L3 device. Does it have MAC addresses? Does it do L1/L2 work?** Yes. Every interface has its own PHY (L1) and MAC controller + MAC address (L2). It receives and strips a frame on the input interface, does the L3 lookup, then ARPs for the next hop and builds a new frame on the output interface. "L3 device" = the highest layer used for forwarding. *Follow-up:* "Can the two sides of a router use different link technologies?" Yes (Wi-Fi in, fibre Ethernet out); IP is the common layer above them.
- **Q: Where is each layer implemented?** App/TLS in user space; TCP/UDP/IP in the kernel (hence syscalls); Ethernet in NIC hardware + driver. *Follow-up [HFT]:* "How do you skip the kernel layers?" Kernel bypass: DPDK/OpenOnload run the transport in user space ([OS 11 §2.9](../Operating%20Systems/11-IO-Systems-IO-Models-and-Disk-Scheduling.md)).
- **Q: Why layering? Any downsides?** Modularity and independent evolution. Downsides: per-layer header overhead, duplicated functions (checksums at several layers), and middleboxes (NAT, firewalls) that break strict layering by peeking at higher layers.
- **Q: What is the bandwidth-delay product and why does it matter?** R × RTT = data in flight needed to keep the pipe full. Windows smaller than BDP waste the link (N4).

### Advanced / killer follow-ups
- **Q [HFT]: We upgraded our New York–Chicago link from 1 to 10 Gbps. Did order latency improve?** Barely. A ~100-byte order's serialization drops from ~0.8 µs to ~80 ns, but propagation (~6.5 ms one way over fibre) dominates. To cut latency, shorten the path or use a faster medium (microwave or laser through air).
- **Q [HFT]: Why does serialization delay still matter in HFT then?** Inside the colocation facility the distances are metres (ns of propagation), so per-hop serialization and switch latency (hundreds of ns) become the dominant terms (N6).
- **Q: Why are messages cut into packets rather than sent whole?** Pipelining across store-and-forward hops (N2); a lost packet means retransmitting only that piece; fair sharing of links; bounded buffer sizes.
- **Q [AMZN]: A service in us-east-1 calls one in ap-south-1 with 50 sequential RPCs. Each RPC is small. What dominates, and how do you fix it?** 50 × RTT (~200 ms) ≈ 10 s of pure propagation. Batch the calls, parallelise them, or move the services closer together. Bandwidth is irrelevant here.
- **Q: Why does queuing delay explode near 100 % utilisation?** With random, bursty arrivals, a queue at intensity → 1 has no spare capacity to drain bursts, so the average wait grows roughly like 1/(1 − ρ). Networks are therefore run well below 100 % to keep latency low.

---

## 5. Common Traps
- Mixing bits and bytes, or powers of 10 and powers of 2 (Mbps is 10⁶ **bits**/s).
- Saying a faster link reduces propagation delay. It reduces only transmission delay.
- Thinking IP addresses change at each router hop. **MAC** addresses change; IPs don't (except NAT).
- Saying a router "forwards the frame". It strips the incoming frame and builds a new one; only the packet travels on (a switch forwards the same frame).
- Thinking a router (an "L3 device") has no L1/L2 or no MAC address. Each interface has its own PHY + MAC; "L3" is just the highest layer it forwards on.
- Forgetting store-and-forward: each hop adds one full L/R.
- Calling the transport unit a "packet" in a precise answer: segment (TCP), datagram (UDP), packet (IP), frame (link).
- Assuming more bandwidth always means faster. Small transfers are latency-bound.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Protocol | format + order + action |
| OSI (7→1) | App, Presentation, Session, Transport, Network, Data link, Physical ("All People Seem To Need Data Processing") |
| TCP/IP | App · Transport (TCP/UDP, port) · Network (IP) · Link (Ethernet/Wi-Fi, MAC) |
| Units | message → segment/datagram → packet → frame → bits |
| Where | app + TLS user space · TCP/UDP/IP kernel · Ethernet NIC + driver |
| Devices | switch L2 (MAC) · router L3 (IP) · L4 LB/NAT/firewall · L7 proxy |
| Per hop | MACs change, IP stays, TTL −1, checksum recomputed |
| Router layers | every interface = own PHY (L1) + MAC address (L2); shared L3 forwarding; link layer is per link, can differ per hop |
| Headers | Eth 14 + FCS 4 (+ 8 preamble + 12 gap) · IPv4 20 · IPv6 40 · TCP 20–60 · UDP 8 |
| Delays | proc + queue + trans (L/R) + prop (d/s ≈ 5 µs/km) |
| Store-and-forward | N packets over k links: (N + k − 1)·L/R |
| Queuing | intensity L·a/R; → 1 ⇒ delay explodes; > 1 ⇒ loss |
| Throughput | = bottleneck link rate |
| BDP | R × RTT = bits in flight; window must be ≥ BDP |
| Latency vs bandwidth | small messages latency-bound; big transfers bandwidth-bound |
