# Computer Networks — Index

> Every company asks networks, but each one asks differently:
> - **Amazon / D. E. Shaw:** concept rounds. Layers, TCP vs UDP, the handshake, congestion control, subnetting, DNS, HTTP, "what happens when you type google.com", load balancers.
> - **HFT (IMC, Optiver, Tower, Graviton, Quadeye):** the *cost* of networking. UDP multicast market data, TCP order entry, `TCP_NODELAY`/Nagle, the Linux receive path, kernel bypass, switch latency, time sync (PTP), microwave vs fibre.
> - **NVIDIA:** data-centre and GPU-cluster networking. RDMA, InfiniBand/RoCE, NVLink, collective communication (all-reduce), NIC offloads.
> - **Qualcomm / TI / Apple (systems, embedded, mobile):** the Linux network stack, sockets, Wi-Fi basics, CRC/checksums, MTU and fragmentation, TCP behaviour on lossy wireless links.
>
> **Depth:** software / system-software roles. Physical-layer signalling (modulation, encoding, line codes) is skipped. Hardware appears only where it affects software (NIC rings, offloads, interrupts). OS-side I/O mechanisms such as `epoll`, `io_uring`, zero-copy and kernel bypass live in [OS 11](../Operating%20Systems/11-IO-Systems-IO-Models-and-Disk-Scheduling.md) and are linked from here, not repeated.

## Study order
| # | Note | Core idea | Time | Must for |
|---|---|---|---|---|
| 01 | [Network Basics & Layered Models](01-Network-Basics-and-Layered-Models.md) | what a network/packet/protocol is, OSI vs TCP/IP, encapsulation, the four delays, bandwidth-delay product | 2 h | **All** |
| 02 | [Data Link Layer & Ethernet](02-Data-Link-Layer-and-Ethernet.md) | framing, MAC addresses, CRC/checksum, Ethernet, switches (MAC learning), ARP, VLAN, MTU, CSMA/CD & Wi-Fi basics | 3 h | All, QCOM, TI |
| 03 | [Reliable Data Transfer & Sliding Window](03-Reliable-Data-Transfer-and-Sliding-Window.md) | stop-and-wait, Go-Back-N, Selective Repeat, link utilisation and sequence-number numericals | 2.5 h | Amazon, DE Shaw |
| 04 | [Network Layer: IP, Subnetting, NAT, IPv6](04-Network-Layer-IP-Subnetting-NAT-IPv6.md) | IPv4 header, CIDR, subnetting numericals, fragmentation, ICMP, DHCP, NAT, IPv6 | 3 h | **All** |
| 05 | [Routing](05-Routing.md) | forwarding vs routing, longest-prefix match, distance vector, link state, BGP, ECMP | 2 h | Amazon, DE Shaw |
| 06 | [Transport Layer: UDP & TCP Connections](06-Transport-Layer-UDP-and-TCP-Connections.md) | ports, the 5-tuple, UDP vs TCP, 3-way handshake, teardown, TIME_WAIT, states, SYN flood | 3 h | **All**, HFT |
| 07 | [TCP Reliability, Flow & Congestion Control](07-TCP-Reliability-Flow-and-Congestion-Control.md) | seq/ack, RTO, fast retransmit, SACK, rwnd, slow start/AIMD, Reno/CUBIC/BBR, Nagle + delayed ACK | 3.5 h | **All**, HFT |
| 08 | [Application Layer: DNS, HTTP, TLS, QUIC](08-Application-Layer-DNS-HTTP-TLS-QUIC.md) | DNS resolution, HTTP/1.1 → 2 → 3, TLS 1.3, QUIC, WebSocket, CDN, "type a URL", common attacks | 3 h | Amazon, Apple, DE Shaw |
| 09 | [Sockets & the Linux Network Stack](09-Sockets-and-the-Linux-Network-Stack.md) | socket API, a packet's path through Linux (NIC ring → IRQ → NAPI → socket), buffers, offloads, socket options, tools | 3 h | **HFT**, NVIDIA, QCOM |
| 10 | [Low-Latency Networking for HFT](10-Low-Latency-Networking-for-HFT.md) | UDP multicast market data, gap recovery, order entry over TCP, switch latency, PTP, microwave vs fibre | 2.5 h | **HFT** |
| 11 | [Data-Centre & GPU-Cluster Networking](11-Data-Center-and-GPU-Cluster-Networking.md) | L4 vs L7 load balancers, anycast, CDN, Clos fabrics, RDMA, RoCE/InfiniBand, NVLink, all-reduce | 2.5 h | **NVIDIA**, Amazon |
| 12 | [Interview Question Bank](12-Interview-Question-Bank.md) | company-wise questions with pointers | ongoing | All |
| 13 | [Cheat Sheet](13-Cheat-Sheet.md) | formulas, numbers, header fields, one-liners | revision | All |

**Total ≈ 33 hours** for a thorough first pass (≈ 2 weeks at 2–3 h/day alongside DSA).

## Links to OS / COA (read these alongside)
| CN topic | Note to recall |
|---|---|
| Interrupts, DMA, NIC hardware | [COA 13 I/O, Interrupts & DMA](../Computer%20Organization%20and%20Architecture/13-IO-Interrupts-DMA.md) |
| Blocking / non-blocking / epoll / io_uring / zero-copy / kernel bypass | [OS 11 I/O Systems & I/O Models](../Operating%20Systems/11-IO-Systems-IO-Models-and-Disk-Scheduling.md) |
| Sockets as IPC, signals (SIGPIPE) | [OS 12 IPC & Signals](../Operating%20Systems/12-IPC-and-Signals.md) |
| IRQ affinity, busy polling, core isolation | [OS 13 Linux Internals & Low-Latency Tuning](../Operating%20Systems/13-Linux-Internals-and-Low-Latency-Tuning.md) |
| Low-latency hardware (NIC, PCIe, FPGA) | [COA 14 Low-Latency Hardware](../Computer%20Organization%20and%20Architecture/14-Low-Latency-Hardware-for-HFT.md) |
| Virtual NICs, SR-IOV, Nitro | [OS 15 Virtualization & Containers](../Operating%20Systems/15-Virtualization-and-Containers.md) |

## How to study each note
1. Read *Intuition* + *Deep Dive*. Every term is defined before it's used; if one isn't, ask Claude and it gets added.
2. Solve the *Worked Numericals* yourself before reading the solution.
3. Answer the *Interview Questions* **out loud**.
4. Run the commands mentioned (`ping`, `traceroute`, `dig`, `ss -tni`, `tcpdump`, `ip addr`). Seeing real packets makes it stick.
5. Ask Claude: *"mock interview me on TCP, Amazon style"* or *"give me 5 subnetting numericals"*.
6. Update the progress table in the root [README](../../README.md).

## Recommended references
| Resource | Use for |
|---|---|
| Kurose & Ross, *Computer Networking: A Top-Down Approach* | standard theory, intuition, end-of-chapter problems |
| Forouzan, *Data Communications and Networking* / Tanenbaum, *Computer Networks* | numericals (CRC, sliding window, subnetting) |
| W. Richard Stevens, *TCP/IP Illustrated Vol. 1* & *UNIX Network Programming Vol. 1* | how TCP really behaves; the socket API |
| Ilya Grigorik, *High Performance Browser Networking* (free online) | latency, TCP/TLS/HTTP/QUIC performance |
| Brendan Gregg, *Systems Performance* (network chapter) | Linux network tools and tuning |
| Cloudflare blog, `man 7 tcp`, `man 7 socket` | modern, practical details |
