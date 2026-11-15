# 13. Computer Networks Cheat Sheet (one-screen revision)

## Formulas
| Topic | Formula |
|---|---|
| Transmission (serialization) delay | L / R |
| Propagation delay | d / s (s ≈ 2 × 10⁸ m/s ⇒ ~5 µs/km fibre; air ~3.34 µs/km) |
| Nodal delay | proc + queue + trans + prop |
| Store-and-forward, N packets over k links | (N + k − 1) · L/R |
| Traffic intensity | L·a / R (→ 1 ⇒ delay explodes; > 1 ⇒ loss) |
| Bandwidth-delay product | R × RTT (bits in flight) |
| Window-limited throughput | window / RTT |
| a (sliding window) | T_p / T_t |
| Stop-and-wait utilisation | 1 / (1 + 2a) |
| Window-N utilisation | min(1, N / (1 + 2a)); full needs N ≥ 1 + 2a |
| Max window, k-bit seq | GBN 2ᵏ − 1 · SR 2ᵏ⁻¹ |
| Subnet | net = A AND mask · bcast = net OR ~mask · hosts = 2^(32−p) − 2 · block = 256 − mask octet |
| Fragment offset | data bytes before / 8; data per fragment = multiple of 8 |
| CSMA/CD min frame | L_min = 2 · d · R / s |
| RTO | SRTT = ⅞·SRTT + ⅛·R · RTTVAR = ¾·RTTVAR + ¼·\|SRTT − R\| · RTO = SRTT + 4·RTTVAR |
| Mathis (loss-limited TCP) | ≈ (MSS / RTT) · 1.22 / √p |
| PTP | delay = ((t2−t1) + (t4−t3))/2 · offset = ((t2−t1) − (t4−t3))/2 |
| Ring all-reduce | per GPU 2(N−1)/N · S; T ≈ 2(N−1)α + 2(N−1)S/(N·B) |
| Packet rate | R / (frame + 20 B preamble/gap) / 8 bits |
| Hamming distance d | detect d − 1, correct ⌊(d − 1)/2⌋ |

## Numbers to quote (typical)
| Thing | Value |
|---|---|
| Ethernet frame | 64 – 1518 B (+ 8 preamble + 12 gap on the wire); MTU 1500; jumbo 9000 |
| Headers | Ethernet 14 + FCS 4 · IPv4 20 (–60) · IPv6 40 · TCP 20 (–60; usually 32 with timestamps) · UDP 8 |
| TCP MSS (IPv4, 1500 MTU) | 1460 (1448 with timestamps) |
| 10 GbE min-frame rate | 14.88 Mpps (67 ns per packet); full frames 813 Kpps |
| Light | ~5 µs/km in fibre; microwave ~3.34 µs/km |
| Chicago ↔ NJ one way | fibre ~6.5 ms; microwave ~3.9–4 ms |
| Intra-DC RTT / cross-continent RTT | ~50–500 µs / ~150–250 ms |
| Kernel UDP receive / busy poll / bypass | ~5 µs / ~2 µs / < 1 µs |
| Cut-through switch / L1 switch | ~100–300 ns / ~5 ns |
| RDMA latency | ~1–2 µs |
| PCIe Gen5 x16 / NVLink 4 (H100) | ~64 GB/s per direction / ~900 GB/s total |
| TCP initial window | 10 MSS (~14.6 KB) |
| Linux TIME_WAIT / min RTO / delayed ACK | 60 s / 200 ms / ~40 ms |
| Linux ephemeral ports | 32768 – 60999 (28,232) |
| Keepalive default | 2 hours |

## Ports and protocol numbers
| | |
|---|---|
| 20/21 FTP · 22 SSH · 25 SMTP · 53 DNS · 67/68 DHCP · 80 HTTP · 123 NTP · 179 BGP · 443 HTTPS (+ QUIC/UDP) · 853 DoT | IP protocol: 1 ICMP · 6 TCP · 17 UDP |
| EtherType: 0x0800 IPv4 · 0x86DD IPv6 · 0x0806 ARP · 0x8100 VLAN | Multicast MAC: 01:00:5e + low 23 bits of group IP |

## Address ranges
| Range | Meaning |
|---|---|
| 10/8, 172.16/12, 192.168/16 | private (NAT) |
| 127/8 · 169.254/16 · 100.64/10 | loopback · link-local · CGNAT |
| 224/4 | multicast |
| IPv6: 2000::/3 · fe80::/10 · ::1 · ff00::/8 · fc00::/7 | global · link-local · loopback · multicast · unique local |
| Mask octets | 128 192 224 240 248 252 254 255 (block 128 64 32 16 8 4 2 1) |

## One-liners per layer
| Layer | Remember |
|---|---|
| Link | CRC detects and drops; switch learns src MAC → port, floods unknown; ARP: broadcast request, unicast reply, remote ⇒ gateway's MAC; CSMA/CD obsolete, Wi-Fi = CSMA/CA + ACKs |
| Network | best effort; LPM; TTL −1 per hop; MAC changes per hop, IP stays (except NAT); DF + PMTUD; DHCP = DORA; NAT = IP:port rewrite table |
| Routing | DV = Bellman-Ford (count-to-infinity; poison reverse) · LS = flood + Dijkstra (OSPF) · BGP = path vector + policy (local-pref > AS-path) · ECMP per-flow hash |
| Transport | UDP 8 B, messages, no state · TCP byte stream, handshake SYN/SYN-ACK/ACK, FIN each way, TIME_WAIT on active closer, CLOSE_WAIT = app bug |
| TCP control | in flight ≤ min(cwnd, rwnd) · slow start ×2/RTT · AIMD +1/÷2 · 3 dup ACKs ⇒ fast retransmit · timeout ⇒ cwnd = 1 · CUBIC default, BBR model-based |
| Latency gotchas | Nagle + delayed ACK = 40 ms ⇒ TCP_NODELAY + one write · HOL blocking ⇒ QUIC/UDP · slow start after idle · bufferbloat |
| App | DNS root → TLD → authoritative, TTL cache, UDP 53 · HTTP methods/status · HTTP/2 multiplex (TCP HOL) · HTTP/3 = QUIC · TLS 1.3 1 RTT, ECDHE + cert chain, forward secrecy |
| Linux stack | NIC (RSS) → DMA ring → IRQ → NAPI softirq → skb/GRO → IP → socket → wake → copy · drops: ring / backlog / socket buffer · TSO/GRO/coalescing trade-offs |
| HFT | multicast A/B + seq gap recovery + stale book · orders TCP tuned · colo, microwave, cut-through/L1, bypass, FPGA · PTP + HW timestamps · p99.9 |
| DC / GPU | L4 vs L7 LB · consistent hashing · anycast + CDN · leaf–spine + ECMP · RDMA (bypass, zero-copy, pinned memory) · IB vs RoCE (PFC/ECN) · NVLink/NVSwitch · ring all-reduce |

## Setup-cost ladder (new connection, until the first response byte)
| Stack | RTTs |
|---|---|
| TCP + TLS 1.2 + request | 4 |
| TCP + TLS 1.3 + request | 3 |
| QUIC (HTTP/3) | 2 |
| QUIC 0-RTT resumption / reused connection | 1 |
