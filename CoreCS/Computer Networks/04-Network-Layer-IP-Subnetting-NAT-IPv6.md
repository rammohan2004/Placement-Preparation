# 04. Network Layer: IP, Subnetting, NAT, IPv6

> **Asked at:** [AMZN] [DESHAW] subnetting numericals, CIDR, "same subnet or via the gateway?", private IPs + NAT, DHCP, how ping/traceroute work, IPv4 vs IPv6, NAT/SNAT port exhaustion · [HFT] multicast address ranges, TTL, MTU/fragmentation (never fragment on the hot path) · [QCOM] [APPLE] IPv6 on mobile (SLAAC, NAT64), DHCP · [NVIDIA] cluster addressing, jumbo-frame MTU and PMTU.
> **Prerequisites:** [01](01-Network-Basics-and-Layered-Models.md) (layers, TTL, IP stays and MAC changes per hop), [02](02-Data-Link-Layer-and-Ethernet.md) (MAC, ARP, MTU, Internet checksum).

---

## 1. Intuition
An IP address is like a postal address with two parts: **city + street** (the *network* part) and **house number** (the *host* part).
- A post office far away looks only at the city and street. It doesn't need to know every house, so one routing entry covers a whole network. That **aggregation** is why the Internet can scale.
- **Subnetting** means deciding where the network part ends and the host part begins.
- **NAT** is like an office receptionist with one public phone number: they connect outside calls to internal extensions using a table.

---

## 2. Deep Dive

### 2.1 What the network layer does
- **Job:** deliver a packet from a source **host** to a destination **host**, across many networks, one router hop at a time.
- **Best effort:** IP promises **nothing**. Packets may be lost, duplicated, reordered or delayed. Reliability is TCP's job (notes 06–07).
- Two parts:
  - **Forwarding (data plane):** each router, for each packet, looks up the destination IP in its **forwarding table** and sends the packet out of the right port. This takes nanoseconds and is done in hardware.
  - **Routing (control plane):** routers exchange information and compute those tables, over seconds to minutes. That's note 05.

### 2.2 IPv4 addresses, prefixes and masks
- An **IPv4 address** is **32 bits**, written as four decimal bytes ("octets"): `192.168.10.77` = `11000000.10101000.00001010.01001101`.
- An address has a **network part** (the first p bits) and a **host part** (the remaining 32 − p bits). The **prefix length** p is written after a slash: `192.168.10.77/26` means the first 26 bits are the network.
- The **subnet mask** is the same information as 32 bits: p ones followed by zeros. /26 = `11111111.11111111.11111111.11000000` = `255.255.255.192`.
- A **subnet** is the set of addresses sharing the same network part. Hosts in the same subnet reach each other directly through the link layer (switch + ARP). Anything else goes through a **router** (the *default gateway*).

**Mask cheat table** (the last non-255 octet):
| Bits set in the octet | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|---|---|---|---|---|---|---|---|
| Octet value | 128 | 192 | 224 | 240 | 248 | 252 | 254 | 255 |
| **Block size** (256 − value) | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |

**The four formulas** (for an address A with prefix p):
```
network address   = A AND mask                 (host bits all 0)
broadcast address = network OR (NOT mask)      (host bits all 1)
total addresses   = 2^(32 − p)
usable hosts      = 2^(32 − p) − 2             (network and broadcast are reserved;
                                                 exceptions: /31 point-to-point links use both, /32 = one host)
```
**Fast method (block size):** in the "interesting" octet (the one where the mask isn't 255 or 0), subnets start at multiples of the block size. The network address is the largest such multiple ≤ the address; the broadcast is the next multiple − 1 (N1).

**How a host decides "direct or via the gateway"** (this ties to ARP, note 02):
```
if (dst_ip AND my_mask) == (my_ip AND my_mask):   same subnet → ARP for dst's MAC, send directly
else:                                             ARP for the default gateway's MAC, send it there
```
Linux shows this with `ip route`: a line like `192.168.10.64/26 dev eth0` (directly connected) plus `default via 192.168.10.65`.

### 2.3 Classful addressing (history) → CIDR
- **Classful** (1981–1993): the first bits fixed the size. **A** = /8 (first octet 1–126), **B** = /16 (128–191), **C** = /24 (192–223), **D** = multicast (224–239), **E** = reserved (240–255). A company needing 2000 addresses had to take a whole class B (65,536), which wasted addresses and bloated routing tables.
- **CIDR** (Classless Inter-Domain Routing, 1993): the prefix can be **any length** (/20, /27, …). This enables:
  1. **Subnetting:** split a block into smaller ones.
  2. **Aggregation (supernetting):** advertise many contiguous blocks as one shorter prefix (N5). This keeps Internet routing tables manageable (~1 M IPv4 routes today instead of hundreds of millions).
- Interviewers still say "class C" to mean a /24. Classes D and E remain meaningful as ranges.

### 2.4 Special addresses to recognise
| Range | Meaning |
|---|---|
| **10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16** | **private** (RFC 1918): free to use inside any organisation, **not routed** on the Internet ⇒ needs NAT to reach outside |
| 127.0.0.0/8 (usually 127.0.0.1) | **loopback**: "this machine"; never leaves the host |
| 169.254.0.0/16 | **link-local**: self-assigned when DHCP fails |
| 0.0.0.0 | "this host / any address" (a server binding to 0.0.0.0 listens on all interfaces) |
| 255.255.255.255 | broadcast on the local network (used by DHCP) |
| **224.0.0.0/4** (224–239) | **multicast** groups (market-data feeds, note 10) |
| 100.64.0.0/10 | carrier-grade NAT space (ISPs) |

### 2.5 The IPv4 header (20 bytes without options)
```
 0                   1                   2                   3   (bits)
 ┌───────┬───────┬───────────────┬───────────────────────────────┐
 │Version│  IHL  │  DSCP  │ ECN  │         Total Length          │
 ├───────┴───────┴───────────────┼─────┬─────────────────────────┤
 │        Identification         │Flags│     Fragment Offset     │
 ├───────────────┬───────────────┼─────┴─────────────────────────┤
 │      TTL      │   Protocol    │        Header Checksum        │
 ├───────────────┴───────────────┴───────────────────────────────┤
 │                       Source IP Address                       │
 ├───────────────────────────────────────────────────────────────┤
 │                    Destination IP Address                     │
 └───────────────────────────────────────────────────────────────┘
```
| Field | Purpose |
|---|---|
| Version | 4 |
| IHL | header length in 4-byte words (5 = 20 B; up to 15 = 60 B with options) |
| DSCP / ECN | DSCP = priority class for QoS. **ECN** (Explicit Congestion Notification) = routers can *mark* a packet "congestion here" instead of dropping it (note 07) |
| **Total Length** | header + data in bytes (max 65,535) |
| Identification, Flags (**DF** = don't fragment, **MF** = more fragments), Fragment Offset | used for fragmentation (§2.6) |
| **TTL** (time to live) | decremented by 1 at each router; at **0 the packet is dropped** and an ICMP "time exceeded" is sent back. Prevents routing loops from circulating packets forever. Typical start: 64 (Linux) or 128 (Windows) |
| **Protocol** | what's inside: **6 = TCP, 17 = UDP, 1 = ICMP** |
| Header Checksum | Internet checksum of the **header only**; recomputed at each hop because TTL changes |
| Source / Destination IP | 32 bits each |

### 2.6 Fragmentation and Path MTU Discovery
If a packet is larger than the next link's **MTU** (note 02), an IPv4 router can **fragment** it: split the data into pieces, each with a copy of the IP header.
- The pieces share the same **Identification**. **Fragment Offset** says where each piece's data starts, **in units of 8 bytes** (so every fragment except the last carries a multiple of 8 data bytes). **MF = 1** on all fragments except the last.
- **Reassembly happens only at the destination host**, never at routers.
- **Why fragmentation is bad:**
  - **one lost fragment ⇒ the whole packet is lost**, and TCP resends all of it;
  - routers do extra work, and the receiver needs reassembly buffers and timeouts;
  - firewalls can't see ports in non-first fragments;
  - fragments are an attack vector.
- **Path MTU Discovery (PMTUD):** TCP sets **DF = 1** (don't fragment). A router that can't forward the packet drops it and sends ICMP *"fragmentation needed, next-hop MTU = X"*. The sender then lowers its packet size. If a firewall blocks those ICMP messages, large packets vanish silently while small ones work (a **PMTU black hole**, a classic "SSH connects but `ls` hangs" bug).
- **IPv6 routers never fragment.** Only the sender may, after PMTUD.
- **[HFT]:** keep messages well under the MTU so they never fragment, which would add latency and turn one loss into a whole-message loss.

### 2.7 ICMP: control and error messages (and how ping/traceroute work)
**ICMP** (Internet Control Message Protocol) carries network-layer messages inside IP packets (protocol 1):
| Message | Sent when |
|---|---|
| Echo request / echo reply | `ping` |
| Destination unreachable (network / host / **port** unreachable / **fragmentation needed**) | no route; nothing listening on a UDP port; packet too big with DF set |
| **Time exceeded** | TTL reached 0 |
| Redirect | a router tells a host about a better first hop |
- **`ping`:** sends echo requests and measures the RTT of each reply. Packets that get no reply count as loss.
- **`traceroute`:** sends probes with **TTL = 1, 2, 3, …**. The router where TTL hits 0 replies "time exceeded" from **its own address**, which reveals the hop and its RTT. Linux sends UDP probes to high ports, so the final host answers "port unreachable" and the trace knows it's done. Windows `tracert` uses ICMP echo. `* * *` means that hop doesn't reply (filtered), not necessarily a problem.

### 2.8 DHCP: how a host gets its address
A new host has no IP. **DHCP** (Dynamic Host Configuration Protocol, runs over UDP ports 67/68) gives it one in four steps, **D-O-R-A**:
```
Client (0.0.0.0)                                     DHCP server
  ── DISCOVER (broadcast to 255.255.255.255) ──────────►   "any DHCP server?"
  ◄──────────────── OFFER  (192.168.1.57, mask, gateway, DNS, lease 24 h) ──
  ── REQUEST (broadcast: "I take 192.168.1.57") ───────►   (tells other servers too)
  ◄──────────────── ACK ──────────────────────────────────
```
The client uses **broadcast** because it has no IP yet and doesn't know the server's address. It gets: IP address, subnet mask, default gateway, DNS servers, and a **lease time** (renewed at half the lease). A **DHCP relay** on the router forwards broadcasts to a central server in another subnet.

### 2.9 NAT: many private hosts behind one public IP
**Problem:** IPv4 has only 2³² ≈ 4.3 billion addresses, and they ran out (the regional registries exhausted their pools 2011–2019). **NAT** (Network Address Translation) lets a whole network of **private** addresses share one or a few **public** addresses.

The usual form is **NAPT / PAT** (port address translation): the NAT router rewrites **source IP + source port** on the way out and remembers the mapping:
```
Inside (private)                    NAT router (public 203.0.113.5)               Internet
192.168.1.10:51000 ── to 93.184.216.34:443 ──►  rewrite src → 203.0.113.5:40001 ──► server
192.168.1.11:51000 ── to 93.184.216.34:443 ──►  rewrite src → 203.0.113.5:40002 ──► server

NAT table:  203.0.113.5:40001 ↔ 192.168.1.10:51000
            203.0.113.5:40002 ↔ 192.168.1.11:51000
Reply to 203.0.113.5:40001 → look up → rewrite dst → 192.168.1.10:51000
```
The router also fixes the IP and TCP/UDP checksums, since the addresses they cover changed.

**Consequences:**
- **Outsiders can't start connections to inside hosts:** no table entry exists for an unsolicited packet. That's a side-effect "firewall". To run a server behind NAT you need **port forwarding** (a static entry).
- **Breaks the end-to-end model:** protocols that write IPs inside their payload (FTP, SIP) need help. Peer-to-peer apps (video calls, games) use **NAT traversal**: **STUN** (learn your public IP:port), **hole punching** (both sides send outward at the same time so both NATs create entries), and **TURN** (relay through a server when nothing else works).
- **Port exhaustion:** one public IP has ~64 K ports, so at most ~64 K simultaneous connections **to the same destination IP:port**. Large services behind NAT gateways or load balancers hit this (**SNAT port exhaustion**, [AMZN]); the fix is more public IPs or connection reuse.
- **CGNAT** (carrier-grade NAT): ISPs put many customers behind shared addresses (100.64.0.0/10), so there are two layers of NAT.

### 2.10 IPv6
| | IPv4 | IPv6 |
|---|---|---|
| Address size | 32 bits (~4.3 × 10⁹) | **128 bits** (~3.4 × 10³⁸) |
| Header | 20–60 B, variable | **fixed 40 B**; optional **extension headers** chained after it |
| Header checksum | yes | **removed** (link CRC + TCP/UDP checksums already cover it; saves per-hop work) |
| Fragmentation by routers | yes | **no**: only the source fragments, after PMTUD; minimum MTU 1280 |
| Broadcast | yes | **none**: multicast is used instead |
| Address configuration | DHCP / manual | **SLAAC** (stateless auto-configuration: the host builds its own address from the router-advertised prefix), or DHCPv6 |
| ARP | ARP | **NDP** (Neighbour Discovery, ICMPv6) |
| NAT | everywhere | not needed (enough addresses); firewalls still used |
| TTL field | TTL | **Hop Limit** (same job) |
| New | — | **flow label** (20 bits) to tag packets of one flow for consistent handling |

**Notation:** 8 groups of 4 hex digits: `2001:0db8:0000:0000:0000:ff00:0042:8329`.
- Drop leading zeros in each group: `2001:db8:0:0:0:ff00:42:8329`.
- Replace **one** run of all-zero groups with `::` ⇒ **`2001:db8::ff00:42:8329`**. Only one `::` is allowed, otherwise the address would be ambiguous.

**Address types:**
| Prefix | Meaning |
|---|---|
| `2000::/3` | global unicast (public); a site typically gets a /48, and a LAN is a **/64** |
| `fe80::/10` | link-local (every interface has one; used by NDP) |
| `::1` | loopback |
| `ff00::/8` | multicast |
| `fc00::/7` | unique local (like private IPv4) |

**Transition:** **dual-stack** (hosts run both; the most common), tunnelling IPv6 inside IPv4, and **NAT64/DNS64** (IPv6-only phones reaching IPv4 servers; Apple requires iOS apps to work on IPv6-only networks [APPLE]).

### 2.11 How to read this note: must know / name-only / skip
| Must know | Name-only | Skip |
|---|---|---|
| network/host split, mask, the 4 formulas, block-size method, same-subnet decision; CIDR + aggregation; VLSM; private/loopback/multicast ranges; IPv4 header fields (TTL, protocol, DF/MF, offset); fragmentation + PMTUD; ping/traceroute mechanics; DHCP DORA; NAPT table + consequences; IPv6 differences and notation | classful ranges; ECN/DSCP details; STUN/TURN; NAT64; flow label | IP options, IPv6 extension-header formats, Mobile IP, IGMP internals (note 10 covers what's needed) |

---

## 3. Worked Numericals

**N1. Subnet details.** Host 192.168.10.77/26.
- Mask: /26 ⇒ the 4th octet has 2 bits set ⇒ 192 ⇒ **255.255.255.192**. Block size = 256 − 192 = **64**.
- Subnets in the 4th octet start at 0, 64, 128, 192. 77 lies in [64, 128) ⇒ **network 192.168.10.64**, **broadcast 192.168.10.127**.
- Usable hosts **.65 – .126** ⇒ 2⁶ − 2 = **62**.
- Binary check: 77 = 01|001101; AND 11|000000 = 01|000000 = 64 ✓.

**N2. Same subnet or via the gateway?** Host 10.1.5.200/22 sends to (a) 10.1.7.3 and (b) 10.1.8.1.
- /22 ⇒ 3rd octet mask 252 (block 4). Networks in the 3rd octet: 0, 4, 8, 12, …
- My network: 5 is in [4, 8) ⇒ **10.1.4.0/22** (covers 10.1.4.0 – 10.1.7.255).
- (a) 10.1.7.3: 7 is in [4, 8) ⇒ **same subnet** ⇒ ARP for 10.1.7.3 directly.
- (b) 10.1.8.1: 8 is in [8, 12) ⇒ **different** ⇒ send to the default gateway.

**N3. Equal-size subnets.** Split 200.10.20.0/24 into at least 6 subnets.
- 6 needs 3 borrowed bits (2³ = 8 ≥ 6) ⇒ /27 ⇒ block 32 ⇒ **8 subnets × 30 usable hosts**.
- 200.10.20.0/27 (.1–.30), .32/27 (.33–.62), .64/27, .96/27, .128/27, .160/27, .192/27, .224/27 (.225–.254).

**N4. VLSM (variable-length subnets).** Allocate 172.16.0.0/24 to LANs needing 100, 50 and 20 hosts, plus one router-to-router link (2 hosts). **Rule: allocate the biggest first**, each at the next boundary that's a multiple of its own block size.
| Need | Smallest fit | Block | Allocated | Usable range |
|---|---|---|---|---|
| 100 | /25 (126) | 128 | 172.16.0.0/25 | .1 – .126 |
| 50 | /26 (62) | 64 | 172.16.0.128/26 | .129 – .190 |
| 20 | /27 (30) | 32 | 172.16.0.192/27 | .193 – .222 |
| 2 | /30 (2) | 4 | 172.16.0.224/30 | .225 – .226 |
Free for later: 172.16.0.228 – .255.

**N5. Route aggregation.** A router has 192.168.4.0/24, 192.168.5.0/24, 192.168.6.0/24 and 192.168.7.0/24. Can it advertise one prefix?
- 3rd octets: 4 = 000001**00**, 5 = 000001**01**, 6 = 000001**10**, 7 = 000001**11**. The common prefix is 000001 = 6 bits ⇒ 16 + 6 = 22.
- ⇒ **192.168.4.0/22**: one routing entry instead of four. (If only .4, .5 and .6 were present, a /22 would also cover .7 and attract traffic for it wrongly.)

**N6. Fragmentation.** A 4000-byte IP datagram (20 B header + 3980 B data, ID = 777) must cross a link with MTU 1500.
- Max data per fragment = 1500 − 20 = 1480 (a multiple of 8 ✓).
| Fragment | Data bytes | Total length | Offset (÷8) | MF |
|---|---|---|---|---|
| 1 | 1480 | 1500 | 0 | 1 |
| 2 | 1480 | 1500 | 1480/8 = **185** | 1 |
| 3 | 3980 − 2960 = 1020 | 1040 | 2960/8 = **370** | 0 |
- If fragment 2 is lost, the destination discards fragments 1 and 3 after its reassembly timeout ⇒ the whole 4000 B is lost.

**N7. NAT capacity.** An office of 2000 users sits behind one public IP. Each user opens ~40 concurrent connections, many to the same popular CDN IP on port 443.
- Connections ≈ 80,000 total. To a **single** destination IP:port, at most ~64,000 source ports are available (fewer in practice: ~28 K with the default Linux ephemeral range 32768–60999).
- If most go to one CDN address ⇒ **port exhaustion** ⇒ new connections fail. Fix: a pool of several public IPs, or fewer, longer-lived connections (HTTP/2 multiplexing).

**N8. IPv6 compression.** Compress `fe80:0000:0000:0000:0204:61ff:fe9d:f156`.
- Drop leading zeros: fe80:0:0:0:204:61ff:fe9d:f156 ⇒ collapse the zero run ⇒ **fe80::204:61ff:fe9d:f156**.

**Try it:** (a) For 172.20.137.45/20, find the mask, network, broadcast and usable host count. (b) Which of 172.20.143.200 and 172.20.144.1 are in the same subnet? (c) Aggregate 10.0.8.0/24 – 10.0.15.0/24.
<details><summary>Answer</summary>(a) /20 ⇒ 255.255.240.0, block 16 in the 3rd octet; 137 is in [128, 144) ⇒ network <b>172.20.128.0</b>, broadcast <b>172.20.143.255</b>, 2¹² − 2 = <b>4094</b> hosts. (b) .143.200 yes; .144.1 no (next block). (c) 8–15 = 00001000–00001111, common prefix 5 bits ⇒ <b>10.0.8.0/21</b>.</details>

---

## 4. Interview Questions

### Basic
- **Q: What is a subnet mask? How many hosts in a /24, /26, /30?** It marks the network bits. Usable hosts = 2^(32−p) − 2 ⇒ 254, 62, 2.
- **Q: What are private IP addresses?** 10/8, 172.16/12, 192.168/16: not routed on the Internet; reused everywhere, reaching outside through NAT.
- **Q: What does TTL do?** Decremented per hop; at 0 the packet is dropped and ICMP time-exceeded is sent ⇒ prevents loops and enables traceroute.
- **Q: How does ping work?** ICMP echo request/reply; it measures RTT and loss.

### Intermediate
- **Q: How does a host decide to send directly vs via the gateway?** Compare (dst AND mask) with (my IP AND mask). Same ⇒ ARP for the destination; different ⇒ ARP for the gateway and send there (N2).
- **Q: Explain CIDR and route aggregation.** Arbitrary prefix lengths; contiguous blocks are advertised as one shorter prefix (N5) ⇒ smaller routing tables, less waste than classful.
- **Q: How does traceroute work?** Probes with TTL 1, 2, 3…; each router where TTL expires replies "time exceeded", revealing itself. It ends at the destination's port-unreachable (UDP) or echo reply.
- **Q: How does NAT work? What breaks?** NAPT rewrites src IP:port and keeps a translation table. Breaks: unsolicited inbound connections, protocols embedding IPs, peer-to-peer (needs STUN/hole punching), and ~64 K ports per public IP per destination.
- **Q: What is DHCP?** DORA over UDP broadcast; gives IP, mask, gateway, DNS and a lease.
- **Q: Why is fragmentation bad, and how does TCP avoid it?** One lost fragment loses the whole packet, plus extra processing and firewall/security problems. TCP sets DF and does Path MTU Discovery, choosing an MSS that fits.

### Advanced / killer follow-ups
- **Q: SSH to a server connects, but large outputs hang. What's going on?** A PMTU black hole: big DF packets are dropped and the "fragmentation needed" ICMP is blocked by a firewall. Fix: allow ICMP type 3 code 4, or clamp the MSS.
- **Q: Why did IPv6 remove the header checksum and router fragmentation?** Faster per-hop forwarding (no checksum recompute when the hop limit changes); the link CRC and transport checksums cover errors; the source handles size via PMTUD.
- **Q: Can two laptops behind different home NATs connect directly?** Usually yes, via STUN + UDP hole punching (both send outward so both NATs create mappings). Symmetric NATs defeat this ⇒ relay via TURN.
- **Q [AMZN]: Thousands of Lambda/EC2 instances behind one NAT gateway call a single third-party API and connections start failing. Why?** SNAT port exhaustion: ~64 K source ports per public IP per destination IP:port. Fix: more NAT IPs, connection pooling/keep-alive, or a VPC endpoint (PrivateLink) that skips NAT.
- **Q: A router has routes 10.0.0.0/8 and 10.1.2.0/24. A packet for 10.1.2.9 arrives. Which route?** The **longest prefix match**: /24 (note 05).
- **Q [HFT]: Which IP range carries market-data multicast, and why does TTL matter there?** 224.0.0.0/4 (exchange groups like 233.x/239.x). TTL limits how far multicast spreads; 224.0.0.x is link-local only (TTL 1).

---

## 5. Common Traps
- Forgetting the −2 for network and broadcast addresses (except /31 and /32).
- Starting a subnet at a non-multiple of its block size (e.g. a /26 at .32 is invalid).
- Allocating VLSM smallest-first (fragments the space). Go biggest first.
- Fragment offset in bytes instead of **8-byte units**.
- Thinking routers reassemble fragments. Only the destination does.
- Thinking NAT is a security feature by design. It's a side effect; use a real firewall.
- Writing two `::` in one IPv6 address.
- Saying IPv6 has broadcast. It uses multicast.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| IP job | host-to-host, best effort; forwarding (data plane) vs routing (control plane) |
| Formulas | net = A AND mask · bcast = net OR ~mask · hosts = 2^(32−p) − 2 |
| Block size | 256 − mask octet; subnets start at multiples |
| Direct vs gateway | (dst & mask) == (me & mask) ? ARP dst : ARP gateway |
| CIDR | any prefix; aggregate contiguous blocks (4 × /24 ⇒ /22) |
| VLSM | biggest first, aligned to own block size |
| Special | private 10/8, 172.16/12, 192.168/16 · 127/8 · 169.254/16 · 224/4 multicast |
| Header | 20 B; TTL; protocol 6/17/1; DF/MF/offset; header checksum per hop |
| Fragmentation | offset ÷ 8; MF; reassemble at destination; avoid via DF + PMTUD |
| ICMP | ping = echo; traceroute = TTL 1, 2, 3 + time exceeded; frag-needed |
| DHCP | Discover, Offer, Request, Ack (broadcast); IP, mask, gateway, DNS, lease |
| NAT | rewrite src IP:port + table; no unsolicited inbound; ~64 K ports per destination |
| IPv6 | 128-bit, 40 B fixed header, no checksum, no router fragmentation, no broadcast, SLAAC, NDP, `::` once |
