# 06. Transport Layer: UDP & TCP Connections

> **Asked at:** [AMZN] [DESHAW] TCP vs UDP, the 3-way handshake (why not 2?), teardown, TIME_WAIT/CLOSE_WAIT, how one port serves thousands of clients, SYN flood, ephemeral-port exhaustion in microservices · [HFT] why order entry is TCP but market data is UDP, message framing on a byte stream, connection setup latency · [QCOM] [APPLE] keepalives vs NAT timeouts on mobile, RST handling · [NVIDIA] connection counts in clusters, per-connection kernel state.
> **Prerequisites:** [01](01-Network-Basics-and-Layered-Models.md) (layers, RTT), [03](03-Reliable-Data-Transfer-and-Sliding-Window.md) (sequence numbers, ACKs, windows), [04](04-Network-Layer-IP-Subnetting-NAT-IPv6.md) (IP addresses, NAT, ICMP). Reliability and congestion details are in [07](07-TCP-Reliability-Flow-and-Congestion-Control.md).

---

## 1. Intuition
IP delivers a packet to a **building** (a host). The transport layer delivers it to the right **apartment** (the process), using the **port number**. It offers two services:
- **UDP is a postcard:** write the address and drop it in the box. No setup, no confirmation, and it may get lost, but it's fast and every postcard stays a separate message.
- **TCP is a phone call:** dial and wait for "hello" (the **handshake**), then talk in a continuous stream. Everything you say arrives, in order, and you hang up politely (**teardown**). Setting up costs time, and the stream has no natural message boundaries.

---

## 2. Deep Dive

### 2.1 Ports, and multiplexing / demultiplexing
- A **port** is a 16-bit number (0–65535) that identifies a communication endpoint **inside a host**.
- **Multiplexing** (sender side): many sockets in many processes send through the one IP layer; each segment is tagged with its source and destination port.
- **Demultiplexing** (receiver side): the kernel reads the ports (and IPs) in the arriving segment and hands it to the right **socket**. A socket is the kernel object an application reads from and writes to, accessed through a file descriptor (note 09).

| Port range | Name | Notes |
|---|---|---|
| 0 – 1023 | **well-known** | binding needs root/`CAP_NET_BIND_SERVICE` on Linux |
| 1024 – 49151 | registered | apps (e.g. 3306 MySQL, 5432 Postgres, 6379 Redis, 8080 alt-HTTP) |
| 49152 – 65535 (IANA) / **32768 – 60999 (Linux default)** | **ephemeral** | auto-assigned to the **client** side of a connection |

Common well-known ports: 20/21 FTP · 22 SSH · 25 SMTP · **53 DNS** · 67/68 DHCP · **80 HTTP** · 123 NTP · **443 HTTPS** · 179 BGP.

**How the kernel finds the socket:**
- **UDP socket:** identified by (destination IP, **destination port**). Datagrams from *any* sender to that port land in the same socket.
- **TCP connection:** identified by the **5-tuple (src IP, src port, dst IP, dst port, protocol)**. A server's **listening socket** on port 443 only accepts new connections. Each accepted client gets its **own connected socket**, distinguished by the client's IP and port:
```
                       server 10.0.0.5, listening on :443
client A 1.2.3.4:51000 ──►  connected socket #1  (1.2.3.4, 51000, 10.0.0.5, 443, TCP)
client A 1.2.3.4:51001 ──►  connected socket #2  (1.2.3.4, 51001, 10.0.0.5, 443, TCP)
client B 5.6.7.8:51000 ──►  connected socket #3  (5.6.7.8, 51000, 10.0.0.5, 443, TCP)
```
So **one server port serves unlimited clients**. Each connection differs in the client part of the tuple. A server's connection count is limited by memory and file descriptors, not by 65,535 ports. Ports run out only on the **client** side, for many connections to the **same** destination (N3).

### 2.2 UDP: User Datagram Protocol
```
 ┌──────────────────┬──────────────────┐
 │  Source port 16  │   Dest port 16   │     8-byte header, that's all
 ├──────────────────┼──────────────────┤
 │   Length 16      │   Checksum 16    │     (checksum optional in IPv4, mandatory in IPv6)
 └──────────────────┴──────────────────┘
```
What UDP gives: ports + checksum. What it **doesn't** give: connection setup, reliability, ordering, flow control, congestion control.
- **Message boundaries are preserved:** one `sendto()` of 100 bytes ⇒ the receiver's `recvfrom()` gets exactly that 100-byte datagram, or nothing if it was lost.
- **Why choose UDP:**
  - **no handshake** (the first packet already carries data);
  - **no head-of-line blocking** (one loss doesn't stall later messages);
  - supports **multicast/broadcast**;
  - the app decides what reliability it wants.
- **Users:** DNS, DHCP, NTP, video and voice calls, games, **market-data feeds [HFT]**, and **QUIC/HTTP-3**, which builds its own reliability on top of UDP (note 08).
- **Size:** max payload 65,507 B over IPv4, but anything over ~1472 B (1500 MTU − 20 IP − 8 UDP) gets fragmented (note 04). So keep datagrams ≤ MTU.

### 2.3 TCP: what it promises
**TCP** (Transmission Control Protocol) gives a **connection-oriented, reliable, in-order, full-duplex byte stream**, with **flow control** (don't overrun the receiver) and **congestion control** (don't overrun the network; note 07).

**The TCP header** (20 bytes + up to 40 bytes of options):
| Field | Purpose |
|---|---|
| Source / destination port | 16 bits each |
| **Sequence number** (32 bits) | the byte number of the **first data byte** in this segment (TCP counts **bytes**, not packets) |
| **Acknowledgement number** (32 bits) | the **next byte expected** from the other side (cumulative: "got everything before this") |
| Data offset | header length in 4-byte words |
| **Flags** | **SYN** (synchronise: start a connection) · **ACK** (the ack field is valid) · **FIN** (I'm done sending) · **RST** (abort now) · PSH (deliver promptly) · URG (rare) · ECE/CWR (ECN congestion signals) |
| **Window** (16 bits) | receive buffer space left, for flow control (scaled by the window-scale option, note 07) |
| Checksum | covers header + data + a "pseudo-header" (src/dst IP), end to end (note 02) |
| **Options** | **MSS** (largest segment I can receive), **window scale**, **SACK permitted**, **timestamps** (RTT measurement + wrap protection); negotiated in the SYNs |

### 2.4 TCP is a byte stream: message framing is *your* job
TCP delivers **bytes in order**, but **not your message boundaries**:
- the sender calls `send("HELLO")` then `send("WORLD")`;
- the receiver's `recv()` might return `"HELLOWOR"` and then `"LD"`, or `"HELLOWORLD"` at once, or `"HE"`.

Segments are cut by MSS, Nagle's algorithm and buffering, not by your calls. So every TCP protocol must **frame** its messages: a length prefix, a delimiter (`\r\n` in HTTP/1.1 headers, the SOH-separated fields of FIX), or a fixed size.
```c
// Read exactly n bytes (a loop is REQUIRED: recv may return fewer)
ssize_t read_n(int fd, char *buf, size_t n) {
    size_t got = 0;
    while (got < n) {
        ssize_t r = recv(fd, buf + got, n - got, 0);
        if (r == 0) return got;          // peer closed (FIN)
        if (r < 0) { if (errno == EINTR) continue; return -1; }
        got += r;
    }
    return got;
}
// Length-prefixed message: read 4-byte length, then the body
uint32_t len; read_n(fd, (char*)&len, 4); len = ntohl(len); read_n(fd, body, len);
```
**[HFT] classic bug:** assuming one `recv()` = one order message. Under load, messages get split or coalesced and the parser breaks.

### 2.5 Opening a connection: the 3-way handshake
```
Client (connect())                                        Server (listen(), accept())
CLOSED                                                    LISTEN
  │── SYN, seq = x, options (MSS, wscale, SACK, TS) ───────►│
SYN_SENT                                                   SYN_RCVD
  │◄──────── SYN + ACK, seq = y, ack = x + 1, options ─────│
ESTABLISHED                                                 │
  │── ACK, seq = x + 1, ack = y + 1  (may carry data) ─────►│
  │                                                        ESTABLISHED → accept() returns
```
- **ISN** (initial sequence number) x and y are chosen **randomly** by each side.
  - Security: an attacker who can't see the traffic can't guess them, so can't inject or reset.
  - Correctness: segments from an **old** connection with the same 5-tuple won't fall inside the new connection's sequence range.
- A **SYN and a FIN each consume one sequence number** (hence x + 1), so they can be ACKed and retransmitted like data.
- **Why 3 messages and not 2?** Each side must (1) tell its ISN and (2) learn that the other side **received** it. Client→server ISN plus its ACK, and server→client ISN plus its ACK, makes 4 messages; the middle two are combined into one SYN-ACK ⇒ 3.
  - With only 2, the server would consider the connection open after sending its SYN-ACK, even when the SYN was an **old, delayed duplicate** from a dead connection. It would allocate resources for a client that doesn't exist and never confirm the client got the server's ISN. The third message proves the client is live and has the server's ISN.
- **Cost:** **1 RTT before any data**, plus the TLS handshake (note 08). That's why connections are **reused** (keep-alive, pools) and why TCP Fast Open and QUIC 0-RTT exist.
- **Inside the server kernel:** a SYN creates an entry in the **SYN queue** (half-open connections). When the final ACK arrives, the connection moves to the **accept queue** (fully established, waiting for the app). `accept()` takes one from there. `listen(fd, backlog)` sets the accept-queue size. If the app is slow to call `accept()`, the queue overflows and new clients see SYN retransmits and timeouts.

### 2.6 Closing a connection: FIN, half-close, RST
```
Client (active closer: calls close() first)                Server (passive closer)
ESTABLISHED                                               ESTABLISHED
  │── FIN, seq = u ────────────────────────────────────────►│
FIN_WAIT_1                                                 CLOSE_WAIT   (read() returns 0 = EOF)
  │◄─────────────────────────────────────── ACK, ack = u+1 ─│
FIN_WAIT_2                                                  │ (server may still SEND data: half-close)
  │◄────────────────────────────────────── FIN, seq = v ────│ server calls close()
TIME_WAIT                                                  LAST_ACK
  │── ACK, ack = v + 1 ────────────────────────────────────►│
  │ (wait 2 × MSL)                                         CLOSED
CLOSED
```
- **4 messages**, because each direction closes **independently**. **Half-close:** `shutdown(fd, SHUT_WR)` sends a FIN ("I'm done sending") while you can still **receive**, e.g. a client that uploads a file and then reads the response. The server's ACK and FIN are often combined (a 3-segment close).
- **RST (reset)** aborts immediately: no FIN dance, and unsent or unread data is discarded. It's sent when:
  - a SYN arrives for a port with no listener ⇒ the client gets **`ECONNREFUSED`**;
  - a segment arrives for a connection the host doesn't know (e.g. after a reboot) ⇒ **"connection reset by peer"** (`ECONNRESET`);
  - an app closes with unread data in its receive buffer, or uses `SO_LINGER` with timeout 0;
  - a firewall or middlebox kills the connection.
- Writing to a connection the peer has reset ⇒ **`SIGPIPE`** / `EPIPE` ([OS 12](../Operating%20Systems/12-IPC-and-Signals.md)). Servers usually ignore SIGPIPE and handle the error code.

### 2.7 The TCP state machine (the states you must know)
| State | Meaning |
|---|---|
| LISTEN | server socket waiting for SYNs |
| SYN_SENT / SYN_RCVD | handshake in progress (client / server) |
| **ESTABLISHED** | open; data flows both ways |
| FIN_WAIT_1 → FIN_WAIT_2 | the **active closer** has sent FIN / got its ACK, and waits for the peer's FIN |
| **CLOSE_WAIT** | the **passive closer** received FIN; **waiting for its own application to call `close()`** |
| LAST_ACK | the passive closer sent its FIN; waiting for the last ACK |
| **TIME_WAIT** | the active closer sent the final ACK; waiting 2 × MSL before forgetting the connection |
| CLOSING | both sides sent FIN simultaneously (rare) |
Use `ss -tan` (Linux) or `netstat -an` to see states. `ss -s` gives counts.

**Debugging by state:**
- **Thousands of `CLOSE_WAIT`** ⇒ **your application bug**: it isn't calling `close()` after reading EOF (leaked sockets).
- **Thousands of `TIME_WAIT`** ⇒ normal for whichever side closes first, but it can exhaust ports (§2.8).
- **Many `SYN_RCVD`** ⇒ a SYN flood or a slow accept loop (§2.9).

### 2.8 TIME_WAIT: why wait, and the problems it causes
The **active closer** stays in TIME_WAIT for **2 × MSL**. MSL is the *maximum segment lifetime*, the assumed longest time a segment can survive in the network. Linux hard-codes the TIME_WAIT duration as **60 s**. Two reasons:
1. **The final ACK might be lost.** The peer would then retransmit its FIN, and TIME_WAIT keeps enough state to **re-send the ACK**. Without it, the reply to that FIN would be an RST, and the peer would see an error.
2. **Old duplicate segments must die.** A delayed segment from this connection could otherwise arrive inside a **new** connection with the **same 5-tuple** and be accepted as valid data. Waiting 2 × MSL guarantees all old segments have expired.

**Problems and fixes:**
- **Client-side port exhaustion:** opening a new connection per request to the same server leaves every used port in TIME_WAIT for 60 s, which caps the connection rate (N3). Fix: **connection pooling / HTTP keep-alive** (reuse connections). On Linux, `net.ipv4.tcp_tw_reuse = 1` lets *outgoing* connections reuse TIME_WAIT ports safely (it relies on timestamps).
- **Server restart: "bind: Address already in use".** The old process's connections are in TIME_WAIT on port 8080. Fix: set **`SO_REUSEADDR`** before `bind()`. Every server should.
- Better design: make the **client** close first, so TIME_WAIT lands on the many clients instead of piling up on the one server.

### 2.9 SYN flood and SYN cookies
- **Attack:** send a huge number of SYNs from spoofed source IPs and never send the final ACK. Each SYN occupies a SYN-queue entry for ~a minute (SYN-ACK retransmissions), the queue fills, and **real clients are refused** (N6).
- **SYN cookies:** when the queue is full, the server **stores nothing**. Instead it encodes the connection's essentials (a hash of the 5-tuple + a secret + a time counter + the MSS) **into its ISN y**. A real client's final ACK carries y + 1, so the server **rebuilds the connection from the ACK** and verifies the hash. Spoofed SYNs cost no memory. Linux enables this automatically under pressure (`net.ipv4.tcp_syncookies = 1`). Cost: some options (e.g. large windows) are limited for those connections.
- Other defences: bigger backlogs, rate limiting, and upstream DDoS scrubbing (AWS Shield, Cloudflare).

### 2.10 Keepalive and idle connections
- **TCP keepalive** (`SO_KEEPALIVE`): after an idle period (Linux default **2 hours**!), send empty probes; no reply after several tries ⇒ the connection is dead. The defaults are far too slow for most apps; tune them with `TCP_KEEPIDLE`/`TCP_KEEPINTVL`/`TCP_KEEPCNT`.
- **NAT and firewall timeouts** (often 30 s – 5 min for idle TCP, less for UDP) silently drop idle mappings, and the next packet then fails or gets an RST. Mobile apps and trading sessions therefore send **application-level heartbeats** (e.g. FIX Heartbeat messages every 30 s) [HFT] [QCOM].

### 2.11 TCP vs UDP
| | TCP | UDP |
|---|---|---|
| Connection | yes (handshake, teardown, state) | no |
| Reliability / ordering | yes (retransmit, reorder) | no |
| Boundaries | **byte stream** (app frames messages) | **messages** preserved |
| Flow + congestion control | yes | no (the app must be careful) |
| Header | 20–60 B | 8 B |
| Setup latency | 1 RTT (+ TLS) | 0 |
| Head-of-line blocking | yes (one loss stalls everything behind it) | no |
| Multicast/broadcast | no | yes |
| Use | web, APIs, DB, SSH, file transfer, **order entry (FIX) [HFT]** | DNS, VoIP/video, games, **market data [HFT]**, QUIC |

**[HFT] why both:** **orders** must arrive exactly once and in order, and the exchange session is long-lived, so **TCP**, tuned (`TCP_NODELAY`, note 07). **Market data** is one-to-many, latency-critical and useless when late, so **UDP multicast** with sequence numbers and gap recovery (note 10).

### 2.12 How to read this note: must know / name-only / skip
| Must know | Name-only | Skip |
|---|---|---|
| ports + ranges; UDP socket vs TCP 5-tuple; listening vs connected socket; UDP header + properties; TCP header fields + flags; byte stream ⇒ framing; handshake with seq/ack numbers + why 3; SYN/accept queues; FIN teardown + half-close; RST cases; states (esp. CLOSE_WAIT, TIME_WAIT); TIME_WAIT reasons + fixes; SYN cookies idea; keepalive vs heartbeats; TCP vs UDP | URG/PSH; simultaneous open/close; TCP Fast Open; `tcp_tw_reuse` details | the full RFC 793 state-transition edge cases, TCP option byte layouts |

---

## 3. Worked Numericals

**N1. Time to first byte.** Client and server are RTT = 100 ms apart. The client requests a small page.
- TCP handshake: **1 RTT**. HTTP request → response: **1 RTT** ⇒ **200 ms**.
- Add TLS 1.3 (1 RTT, note 08) ⇒ **300 ms**. TLS 1.2 (2 RTTs) ⇒ **400 ms**.
- Reusing an existing connection ⇒ **100 ms**. This is why keep-alive and connection pools matter.

**N2. Sequence and ACK numbers.** The client's ISN is 1000 and the server's is 5000. After the handshake, the client sends 3 segments of 500 bytes, then closes.
| Segment | seq | ack | Note |
|---|---|---|---|
| C → S SYN | 1000 | — | SYN uses 1 number |
| S → C SYN-ACK | 5000 | 1001 | |
| C → S ACK | 1001 | 5001 | |
| C → S data #1 | 1001 | 5001 | bytes 1001–1500 |
| C → S data #2 | 1501 | 5001 | bytes 1501–2000 |
| C → S data #3 | 2001 | 5001 | bytes 2001–2500 |
| S → C ACK | 5001 | **2501** | cumulative: all 1500 bytes |
| C → S FIN | 2501 | 5001 | FIN uses 1 number |
| S → C ACK | 5001 | **2502** | |

**N3. TIME_WAIT port exhaustion.** A service opens a **new** TCP connection per request to one backend (same dst IP:port), and it closes first. Linux ephemeral range 32768–60999.
- Ports available = 60999 − 32768 + 1 = **28,232**. Each stays in TIME_WAIT for 60 s.
- Max sustained rate = 28,232 / 60 ≈ **470 new connections/s**. Beyond that, `connect()` fails with `EADDRNOTAVAIL`.
- With a pool of 50 persistent connections, each serving ~1000 requests/s ⇒ **50,000 requests/s**, with no new connections at all.

**N4. Sequence-number wraparound.** The 32-bit sequence space = 2³² bytes ≈ 4.29 GB.
- At 1 Gbps: wraps in 4.29 × 10⁹ × 8 / 10⁹ ≈ **34 s**. At 100 Gbps: **0.34 s**, which is far shorter than the MSL (60 s+). An old delayed segment could then carry a *valid-looking* sequence number.
- Fix: **PAWS** (Protection Against Wrapped Sequences), which uses the **timestamp option** as an extra 32-bit, always-increasing tag. Fast links need timestamps enabled.

**N5. Header overhead for small messages.** A 50-byte message over IPv4.
- UDP: 50 / (50 + 8 + 20) = **64 %**.
- TCP with the usual 12-byte timestamp option: 50 / (50 + 32 + 20) = **49 %** (before Ethernet's 38 bytes of framing).

**N6. SYN flood sizing.** An attacker sends 10,000 spoofed SYNs/s. Linux retries the SYN-ACK 5 times (1 + 2 + 4 + 8 + 16 + 32 s ≈ 63 s before giving up).
- Half-open entries needed = 10,000 × 63 ≈ **630,000**. Typical `tcp_max_syn_backlog` is in the thousands ⇒ the queue is full within a second ⇒ legitimate SYNs are dropped.
- SYN cookies ⇒ **0** stored entries; the server just computes the cookie per SYN.

**Try it:** (a) The client ISN is 7000 and the server ISN is 300. The client sends 2 segments of 1000 B; the server replies with 1 segment of 400 B. Give the ack number in the server's ACK of the client data, and the seq of the server's data segment. (b) A client must make 2000 new connections/s to one server and closes first. Will the default Linux ephemeral range survive TIME_WAIT? What's the fix?
<details><summary>Answer</summary>(a) Server ACK = 7001 + 2000 = <b>9001</b>; server data seq = <b>301</b> (its SYN used 300). (b) Max ≈ 28,232/60 ≈ 470/s < 2000 ⇒ <b>no</b>. Fix: connection reuse/pooling (best), widen the ephemeral range, enable <code>tcp_tw_reuse</code>, or have the server close first.</details>

---

## 4. Interview Questions

### Basic
- **Q: TCP vs UDP?** §2.11 table. Reliable ordered byte stream with connection, flow and congestion control, vs connectionless best-effort messages with an 8-byte header.
- **Q: What is a port? How does the OS know which process gets a packet?** A 16-bit endpoint ID. UDP: by destination IP:port. TCP: by the 5-tuple, matched to a connected socket (or the listening socket for a new SYN).
- **Q: Explain the 3-way handshake.** SYN (seq x) → SYN-ACK (seq y, ack x+1) → ACK (ack y+1). Exchanges ISNs and options (MSS, window scale, SACK, timestamps).

### Intermediate
- **Q: Why 3-way and not 2-way?** Both ISNs must be sent and confirmed. With 2, the server can't tell an old duplicate SYN from a live client, and never learns that the client got its ISN (§2.5).
- **Q: Why are ISNs random?** Security (blind injection/reset needs the sequence number) and correctness (old segments with the same 5-tuple won't land in the new connection's window).
- **Q: What is TIME_WAIT? Who enters it and why 2 × MSL?** The active closer, after sending the last ACK. It re-ACKs a retransmitted FIN and lets old duplicates expire. Linux uses 60 s.
- **Q: How can a server on port 443 handle 100,000 clients when ports are 16-bit?** Connections are identified by the 5-tuple; clients differ by IP:port. The limits are memory, fds and CPU, not ports.
- **Q: I see thousands of CLOSE_WAIT sockets. What's wrong?** The app received FIN (read returned 0) but never called `close()`: leaked sockets in your code.
- **Q: What happens when you connect to a port with no listener?** The host replies with RST ⇒ `ECONNREFUSED`. (If a firewall silently drops the SYN instead, `connect()` times out.)

### Advanced / killer follow-ups
- **Q [HFT]: Your FIX parser occasionally gets garbage under heavy load. Why?** TCP is a byte stream: `recv()` returns partial or coalesced messages. Frame properly (length prefix / delimiter + an accumulation buffer, §2.4).
- **Q [AMZN]: A microservice making HTTP calls starts failing with `EADDRNOTAVAIL` at high QPS. Why, and how do you fix it?** Ephemeral ports stuck in TIME_WAIT from per-request connections (N3). Use keep-alive/pooling; possibly `tcp_tw_reuse`, a wider port range, more destination IPs.
- **Q: What is a SYN flood and how do SYN cookies work?** Spoofed SYNs exhaust the half-open queue. Cookies encode the connection state in the server's ISN, so nothing is stored until a valid ACK returns (§2.9).
- **Q: Why does a restarted server fail with "Address already in use", and how do you fix it?** Old connections are in TIME_WAIT on that port. Set `SO_REUSEADDR` before `bind()`.
- **Q [HFT]: Why is order entry TCP while market data is UDP multicast?** Orders need exactly-once, in-order delivery over a long session. Market data is one-to-many, latency-critical and stale when late ⇒ multicast + sequence numbers + gap recovery, no retransmission stalls.
- **Q: The peer's machine lost power mid-connection. What does your side see?** Nothing until you send (then retransmissions time out, minutes later), or until keepalive (2 h default) or app heartbeats detect it. When the peer reboots and gets your segment, it replies RST. Hence heartbeats.
- **Q: FIN vs RST?** FIN is a graceful, per-direction close (all data delivered). RST aborts immediately and discards buffered data; the peer gets ECONNRESET.

---

## 5. Common Traps
- Treating TCP as message-based (one `send` = one `recv`). It's a byte stream.
- Thinking the server enters TIME_WAIT. The **active closer** does, whichever side closes first.
- Thinking a server is limited to ~65 K connections because of ports.
- Forgetting that SYN and FIN consume a sequence number (ack = seq + 1).
- Confusing CLOSE_WAIT (your app hasn't closed) with TIME_WAIT (normal, after closing).
- Forgetting `SO_REUSEADDR` on servers.
- Sending UDP datagrams larger than the MTU (they fragment; one lost fragment loses the whole datagram).

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Ports | 0–1023 well-known · ephemeral 32768–60999 (Linux) · 53 DNS, 80, 443, 22 |
| Demux | UDP: dst IP:port · TCP: 5-tuple; listen socket + one connected socket per client |
| UDP | 8 B header; no connection/reliability/order/congestion control; boundaries kept; multicast |
| TCP | reliable, ordered, full-duplex **byte stream**; flow + congestion control |
| Header | seq (first byte), ack (next expected), flags SYN/ACK/FIN/RST, window, options MSS/wscale/SACK/TS |
| Framing | app adds length prefix/delimiter; loop on recv |
| Handshake | SYN x → SYN-ACK y, x+1 → ACK y+1; 1 RTT; random ISN; SYN queue → accept queue |
| Why 3 | both ISNs sent + confirmed; reject old duplicate SYNs |
| Close | FIN/ACK each way (half-close possible); RST = abort |
| States | CLOSE_WAIT = app didn't close · TIME_WAIT = active closer, 2·MSL (60 s) |
| TIME_WAIT | re-ACK lost FIN, flush old duplicates; ports ⇒ pool connections; server `SO_REUSEADDR` |
| SYN flood | half-open exhaustion; SYN cookies encode state in ISN |
| Keepalive | 2 h default; NAT timeouts ⇒ app heartbeats |
| HFT | orders TCP (+ NODELAY) · market data UDP multicast |
