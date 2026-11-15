# 12. Computer Networks Interview Question Bank (company-wise)

> How to use: answer **out loud** in 1–2 minutes, then check the linked note. ⭐ = very frequently asked. Pointers are note §section.
> For a mock interview: *"mock interview me on networks, <company> style"*.

---

## 1. Universal top 25 (every company)
| # | Question | Where |
|---|---|---|
| 1 ⭐ | **What happens when you type google.com and press Enter?** | 08 §2.9 |
| 2 ⭐ | OSI vs TCP/IP layers; what each does; unit and address at each layer | 01 §2.4 |
| 3 ⭐ | TCP vs UDP; when would you use each? | 06 §2.11 |
| 4 ⭐ | Explain the 3-way handshake. Why not 2-way? | 06 §2.5 |
| 5 ⭐ | Connection teardown; what is TIME_WAIT and why 2·MSL? | 06 §2.6–2.8 |
| 6 ⭐ | Flow control vs congestion control | 07 §2.2–2.3 |
| 7 ⭐ | Slow start, congestion avoidance, fast retransmit/recovery; draw cwnd | 07 §2.3, §3 N1 |
| 8 ⭐ | How does DNS resolution work? Recursive vs iterative; record types | 08 §2.2 |
| 9 ⭐ | Subnetting: network, broadcast, host count for a.b.c.d/p | 04 §2.2, §3 |
| 10 ⭐ | What is NAT and what does it break? | 04 §2.9 |
| 11 ⭐ | Hub vs switch vs router | 02 §2.5, 01 §2.4 |
| 12 ⭐ | What is ARP? What happens for a remote destination? | 02 §2.8 |
| 13 | MAC vs IP address: why do we need both? | 02 §2.2 |
| 14 | HTTP/1.1 vs HTTP/2 vs HTTP/3 | 08 §2.4 |
| 15 | HTTPS/TLS handshake; how is a certificate verified? | 08 §2.6 |
| 16 | GET vs POST; idempotency; key status codes | 08 §2.3 |
| 17 | Transmission vs propagation delay; bandwidth-delay product | 01 §2.6–2.7 |
| 18 | Stop-and-wait vs Go-Back-N vs Selective Repeat; window-size limits | 03 §2.6 |
| 19 | Distance vector vs link state; count-to-infinity | 05 §2.3–2.5 |
| 20 | Longest-prefix match | 05 §2.1 |
| 21 | How do ping and traceroute work? | 04 §2.7 |
| 22 | DHCP: how does a machine get an IP? | 04 §2.8 |
| 23 | IPv4 vs IPv6 | 04 §2.10 |
| 24 | How can one server port handle thousands of clients? | 06 §2.1 |
| 25 | CRC vs checksum; why checksums at several layers? | 02 §2.4 |

---

## 2. Amazon [AMZN] (SDE: concepts + practical + a bit of system design)
- ⭐ Type-a-URL walkthrough, with drill-downs: "what if DNS fails?", "where does the TLS cert come from?", "why is the second load faster?" (08 §2.9)
- ⭐ TCP vs UDP for a video-call app / a chat app / a file upload (06 §2.11, 08 §2.7)
- ⭐ L4 vs L7 load balancer; which algorithm; health checks (11 §2.1)
- Consistent hashing for a cache fleet: what happens when a node is added? (11 §2.1, N3)
- A service making many short HTTP calls fails with `EADDRNOTAVAIL`: why? (06 §2.8, N3)
- Lambdas behind a NAT gateway can't open new connections to one API: why? (04 §2.9, N7)
- How does Route 53 latency-based routing work? DNS vs anycast (08 §2.2, 11 §2.2)
- Why is a cross-region chatty service slow? (01 §4 RTT question)
- How does a CDN speed up a page? (08 §2.5, 11 §2.2)
- Thousands of CLOSE_WAIT sockets on a service: what's wrong? (06 §2.7)
- HTTP status codes in an incident: 502 vs 503 vs 504 (08 §2.3)
- Design a reliable protocol over UDP (03 §4)

## 3. D. E. Shaw [DESHAW] (precise reasoning, numericals, coding)
- ⭐ Subnetting + VLSM + aggregation numericals (04 §3)
- ⭐ Sliding-window utilisation and sequence-number bits (03 §3)
- ⭐ Implement longest-prefix match (trie) (05 §4)
- CRC computation by hand (02 §3 N1)
- Why does GBN need N ≤ 2ᵏ − 1 and SR N ≤ 2ᵏ⁻¹? (03 §2.6, N5)
- Why is the TCP ISN random? Why 3 duplicate ACKs? (06 §2.5, 07 §2.1)
- Dijkstra trace on a small graph (05 §3 N2)
- AIMD fairness argument (07 §2.3)
- Diffie–Hellman by hand; what is forward secrecy? (08 §3 N5)

## 4. HFT: IMC, Optiver, Tower, Graviton, Quadeye [HFT]
- ⭐ Why is market data UDP multicast while order entry is TCP? (10 §2.2, §2.4)
- ⭐ How do you detect and recover from a lost market-data packet? A/B arbitration (10 §2.3)
- ⭐ What happens in Linux from the NIC to `recv()`? Where does the time go? (09 §2.4, N6)
- ⭐ Nagle + delayed ACK: why do small messages take 40 ms? `TCP_NODELAY` (07 §2.5)
- Where can UDP packets be dropped on your host and how do you find out? (09 §2.8)
- How do you reduce kernel network latency without full bypass? (09 §4, OS 13)
- Kernel bypass: DPDK vs OpenOnload vs AF_XDP (OS 11 §2.9, 09 §2.10)
- Microwave vs fibre; why doesn't more bandwidth help NY–Chicago? (10 §2.6, 01 §4)
- Serialization delay of an 84-byte frame at 10/25 Gbps (01 §3 N6)
- Cut-through vs store-and-forward switches (02 §2.7)
- PTP vs NTP; compute the offset from 4 timestamps (10 §2.7, N3)
- What is a microburst? (10 §2.8)
- TCP is a byte stream: how do you frame FIX/binary messages? (06 §2.4)
- Slow start after idle at the market open (07 §2.6)
- Design a feed handler (10 §4)
- Tick-to-trade budget breakdown (10 §3 N4)

## 5. NVIDIA [NVIDIA] (systems software, clusters, drivers)
- ⭐ RDMA: what it is, why it's faster than TCP, memory registration (11 §2.5)
- ⭐ InfiniBand vs RoCE; why RoCE needs lossless Ethernet; PFC problems (11 §2.5)
- ⭐ Ring all-reduce: communication cost and time (11 §2.7, N5)
- NVLink/NVSwitch vs PCIe vs InfiniBand (11 §2.6)
- GPUDirect RDMA (11 §2.6)
- Packet rates at 100 Gbps: why one core can't keep up (09 §3 N1, §4)
- NIC offloads: TSO, GRO, checksum; RSS (09 §2.6–2.7)
- ECMP and elephant-flow collisions in training fabrics (05 §2.7, N5)
- TCP incast (07 §2.6)
- Bandwidth-delay product and window sizing (01 §2.7, 07 §2.2)

## 6. Qualcomm / TI [QCOM] [TI] (embedded / mobile / drivers)
- ⭐ CRC computation and why it's done in hardware (02 §2.4)
- ⭐ CSMA/CD vs CSMA/CA; hidden terminal; why Wi-Fi uses link-layer ACKs (02 §2.6)
- NAPI and interrupt mitigation in network drivers (09 §2.4, OS 11 §2.2)
- MTU, fragmentation, PMTUD (04 §2.6)
- TCP over lossy wireless links: why it underperforms (07 §2.6)
- Keepalive vs NAT timeouts on mobile (06 §2.10)
- Socket API in embedded Linux; byte order (09 §2.2)
- IPv6 on mobile networks: SLAAC, NAT64 (04 §2.10)

## 7. Apple [APPLE]
- ⭐ HTTP/3 + QUIC: benefits, connection migration (08 §2.7)
- ⭐ TLS 1.3 handshake; certificate pinning; App Transport Security (HTTPS required) (08 §2.6, §2.10)
- DNS resolution and caching on a device (08 §2.2)
- IPv6-only networks (NAT64/DNS64) (04 §2.10)
- Wi-Fi jitter and loss effects on apps (02 §2.6, 07 §2.6)

---

## 8. Rapid-fire (answer in one line)
1. Port for DNS / HTTPS / SSH? → 53 / 443 / 22
2. Size of the IPv4 / IPv6 / TCP / UDP / Ethernet headers? → 20 / 40 / 20–60 / 8 / 14 + 4 FCS
3. Ethernet MTU and TCP MSS? → 1500 / 1460
4. Min/max Ethernet frame? → 64 / 1518 bytes
5. Usable hosts in a /27? → 30
6. Private IPv4 ranges? → 10/8, 172.16/12, 192.168/16
7. Who enters TIME_WAIT? → the active closer
8. What does TTL prevent? → infinite routing loops
9. What does a switch use to forward, and a router? → MAC / IP (longest-prefix match)
10. Fast retransmit threshold? → 3 duplicate ACKs
11. Slow-start growth per RTT? → doubles
12. What does 304 mean? → Not Modified (use your cached copy)
13. What does IGMP do? → join/leave multicast groups
14. Why random source ports for DNS queries? → resist cache poisoning
15. GBN max window with 3-bit sequence numbers? → 7 (SR: 4)
16. Speed of light in fibre per km? → ~5 µs/km
17. Packets/s at 10 Gbps with minimum frames? → 14.88 M
18. TLS 1.3 handshake RTTs? → 1 (0-RTT on resumption)
19. Protocol numbers in the IP header for TCP/UDP/ICMP? → 6 / 17 / 1
20. What does `SO_REUSEADDR` solve? → bind while old connections are in TIME_WAIT
