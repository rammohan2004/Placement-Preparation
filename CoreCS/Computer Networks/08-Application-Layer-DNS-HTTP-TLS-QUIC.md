# 08. Application Layer: DNS, HTTP, TLS, QUIC

> **Asked at:** [AMZN] **"What happens when you type google.com and press Enter?"** (the most-asked networking question), DNS resolution and record types, HTTP methods/status codes, HTTP/1.1 vs 2 vs 3, Route 53 DNS routing · [DESHAW] TLS handshake, idempotency, caching · [APPLE] HTTPS everywhere (App Transport Security), HTTP/3 + QUIC on iOS, connection migration · [HFT] why trading doesn't use HTTP on the hot path; FIX over TCP · [NVIDIA] [QCOM] gRPC/HTTP-2 in services, TLS offload.
> **Prerequisites:** [04](04-Network-Layer-IP-Subnetting-NAT-IPv6.md) (IP, NAT, anycast basics), [06](06-Transport-Layer-UDP-and-TCP-Connections.md) (TCP/UDP, handshake), [07](07-TCP-Reliability-Flow-and-Congestion-Control.md) (slow start, head-of-line blocking).

---

## 1. Intuition
To visit a shop, you first look up its address in a phone book (**DNS**: name → IP address). You walk in and talk in an agreed language (**HTTP**: "GET me this page"). If you don't want eavesdroppers, you first check the shopkeeper's ID card, which a trusted authority stamped (a **certificate**), and agree on a secret code (**TLS**). **QUIC** is a newer way to do all of this faster: the ID check, the secret code and the conversation get set up together, and one dropped sentence doesn't make everyone wait.

---

## 2. Deep Dive

### 2.1 Application-layer basics
- An **application-layer protocol** defines the messages two programs exchange over a socket: their format, order and meaning. It runs in **user space**, on top of TCP or UDP.
- **Client–server:** an always-on server with a known address; clients start requests (web, email, APIs). **Peer-to-peer:** peers are both clients and servers (BitTorrent, parts of video calls).

### 2.2 DNS: Domain Name System
**Job:** translate human names (`www.amazon.in`) into IP addresses (and other records). It's a **distributed, hierarchical, cached database**. No single server knows everything.

**The hierarchy** (read a name right to left):
```
                         . (root)  — 13 named root server identities, hundreds of anycast sites
             ┌──────────┼───────────┐
           .com        .in         .org        ← TLD (top-level domain) servers
             │          │
        amazon.com   amazon.in                 ← authoritative servers for each domain (run by the owner or its DNS provider)
             │
    www.amazon.com → A 52.94.236.248           ← the actual record
```
**The players:**
- **Stub resolver:** the small resolver library in your OS. It asks one server and waits.
- **Recursive resolver:** your ISP's or a public one (8.8.8.8, 1.1.1.1). It does the whole hunt for you and **caches** the results.
- **Root / TLD / authoritative servers:** each knows only its own part of the tree and refers the resolver downward.

**Resolution step by step** (cold cache):
```
Browser cache? OS cache (and /etc/hosts)? → miss
1. Stub → recursive resolver:  "www.amazon.com?"                  (recursive query: "give me the final answer")
2. Resolver → root:            "www.amazon.com?" → "ask .com servers (here are their IPs)"     (iterative referral)
3. Resolver → .com TLD:        → "ask amazon.com's servers: ns1.amazon.com …"
4. Resolver → authoritative:   → "www.amazon.com A 52.94.236.248, TTL 60"
5. Resolver caches it for TTL seconds and answers the stub; the OS and browser cache it too
```
- **Recursive vs iterative:** the stub asks recursively ("you do the work"). The resolver queries the hierarchy **iteratively**: each server answers with a **referral** ("ask them"), not the final answer.
- **Caching + TTL:** every record carries a **TTL** (time to live, in seconds), and caches keep it that long. TLD referrals are cached too, so most lookups need only the last step, or none at all. "DNS propagation delay" just means old cached records waiting out their TTL. Lower the TTL **before** a planned migration.
- **Transport:** **UDP port 53** for normal queries: one small request and one small reply, with no handshake. **TCP 53** when the answer is too large (truncated flag) and for zone transfers. Privacy variants: **DoT** (DNS over TLS, port 853) and **DoH** (DNS over HTTPS).

**Record types:**
| Type | Maps | Example / use |
|---|---|---|
| **A** | name → IPv4 | `www.example.com A 93.184.216.34` |
| **AAAA** | name → IPv6 | |
| **CNAME** | alias → canonical name | `www.shop.com CNAME shop.cdn.net` (CDNs use this) |
| **MX** | domain → mail server (with priority) | email delivery |
| **NS** | domain → its authoritative name servers | delegation |
| **TXT** | free text | domain verification, SPF/DKIM email anti-spoofing |
| **PTR** | IP → name (reverse lookup) | logs, mail-server checks |
| SOA | zone metadata (primary server, serial, negative-cache TTL) | |
| SRV | service → host + port | service discovery (SIP, some Kubernetes uses) |

**DNS as a traffic tool [AMZN]:** an authoritative server can return **different answers per client**: round-robin over several IPs, **geo / latency-based** (send India users to Mumbai), **weighted** (canary 5 %), **failover** (drop unhealthy IPs). AWS **Route 53** does all of these. The limits: clients cache answers (so the TTL bounds how fast changes take effect), and the server sees the *resolver's* location, not the user's (EDNS Client Subnet helps).

**Attacks:**
- **Cache poisoning:** forge a reply that reaches the resolver before the real one, so it caches the attacker's IP. Defences: random query IDs + **random source ports** (attackers must guess ~2³² combinations), and **DNSSEC** (signed records).
- **DNS amplification DDoS:** send small queries with the **victim's spoofed IP**; open resolvers send large replies to the victim (N4).

### 2.3 HTTP basics
**HTTP** (HyperText Transfer Protocol) is a **request/response**, text-based (in 1.1), **stateless** protocol.
```
GET /search?q=tcp HTTP/1.1                 ← request line: method, path, version
Host: www.google.com                       ← headers (Host is mandatory in 1.1: many sites share one IP)
User-Agent: Mozilla/5.0
Accept: text/html
Cookie: session=abc123
                                           ← blank line ends the headers; a body would follow for POST/PUT

HTTP/1.1 200 OK                            ← status line
Content-Type: text/html; charset=utf-8
Content-Length: 5123                       ← framing: how many body bytes (or chunked encoding)
Cache-Control: max-age=60
Set-Cookie: session=abc123; Secure; HttpOnly

<html>…
```
**Methods** (*safe* = doesn't change server state; *idempotent* = repeating it has the same effect as doing it once):
| Method | Use | Safe | Idempotent |
|---|---|---|---|
| GET | read a resource | ✅ | ✅ |
| HEAD | GET without the body (check headers) | ✅ | ✅ |
| POST | create / submit / trigger an action | ❌ | ❌ |
| PUT | replace a resource at a known URL | ❌ | ✅ |
| PATCH | partially modify | ❌ | ❌ (usually) |
| DELETE | delete | ❌ | ✅ |
| OPTIONS | ask what's allowed (CORS preflight) | ✅ | ✅ |

Idempotency matters for **retries**: a client may safely retry a timed-out PUT or DELETE, but retrying a POST can double-charge a payment. APIs therefore use **idempotency keys**.

**Status codes:**
| Class | Meaning | Must-know codes |
|---|---|---|
| 1xx | informational | 101 Switching Protocols (WebSocket upgrade) |
| 2xx | success | **200 OK**, 201 Created, 204 No Content |
| 3xx | redirect / cache | **301 Moved Permanently**, **302 Found** (temporary), **304 Not Modified** |
| 4xx | **client** error | 400 Bad Request, **401 Unauthorized** (not authenticated), **403 Forbidden** (authenticated but not allowed), **404 Not Found**, **429 Too Many Requests** (rate limited) |
| 5xx | **server** error | **500 Internal Server Error**, **502 Bad Gateway** (the proxy/LB got a bad response from upstream), **503 Service Unavailable** (overloaded/maintenance), **504 Gateway Timeout** (the upstream was too slow) |

**State over a stateless protocol:** each request stands alone. Sessions use **cookies** (the server sets an ID; the browser sends it back on every request) or **tokens** (`Authorization: Bearer <JWT>`).

### 2.4 HTTP versions
| | HTTP/1.0 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|---|
| Connection use | new TCP connection **per request** | **persistent** (keep-alive) by default | **one** TCP connection, many **streams** | **QUIC** over UDP |
| Concurrency | none | one request at a time per connection (pipelining existed but was broken and disabled) ⇒ browsers open **~6 connections per host** | **multiplexed** streams: requests and responses interleaved as frames | multiplexed independent streams |
| Format | text | text | **binary frames** | binary frames |
| Header compression | no | no | **HPACK** | QPACK |
| Head-of-line blocking | — | at the HTTP level (request queue) | HTTP level solved; **TCP-level HOL remains** (one lost packet stalls all streams) | **solved**: a loss stalls only its own stream |
| Setup cost (new, with TLS 1.3) | TCP + TLS + request | same | same | **1 RTT** (0-RTT on resume) |

### 2.5 Caching and conditional requests
- **`Cache-Control: max-age=3600`** ⇒ the browser/CDN reuses the response for an hour without asking. `no-store` ⇒ never cache. `private` ⇒ browser only, not shared caches.
- **Validation:** the response carries an **ETag** (a version tag) or `Last-Modified`. Later the client asks `If-None-Match: "v7"`, and if nothing changed the server replies **304 Not Modified** with no body: one RTT, no transfer.
- **CDN** (Content Delivery Network): caches copies at edge servers close to users, reached through DNS/anycast (note 11). It cuts RTT, which is what dominates small transfers (note 07 N2).

### 2.6 TLS: making the channel secure
**Goals:** **confidentiality** (eavesdroppers see gibberish), **integrity** (tampering is detected), **authentication** (you're really talking to `amazon.in`).

**Building blocks** (just enough to explain the handshake):
| Block | What it is |
|---|---|
| **Symmetric encryption** (AES-GCM, ChaCha20) | both sides share **one secret key**; very fast ⇒ used for all the bulk data. Problem: how do you agree on the key over a public network? |
| **Asymmetric (public-key) crypto** (RSA, elliptic curves) | a **key pair**: the public key can be shared; only the private key can decrypt or **sign**. Slow ⇒ used only in the handshake |
| **(EC)Diffie–Hellman key exchange** | both sides send public values and each computes the **same shared secret**; an eavesdropper who sees both public values can't compute it (N5) |
| **Hash / MAC** | a fingerprint of data; with a key (HMAC/AEAD tag) it proves the data wasn't modified |
| **Certificate** | a document saying "this public key belongs to `amazon.in`", **signed by a Certificate Authority (CA)** |
| **Chain of trust** | server cert ← signed by an intermediate CA ← signed by a **root CA** whose certificate is pre-installed in your OS/browser. The client verifies each signature, the name (and SAN list), and the expiry date |
| **Forward secrecy** | fresh ephemeral DH keys per connection ⇒ stealing the server's private key **later** can't decrypt recorded old traffic. TLS 1.3 makes this mandatory |

**TLS 1.3 handshake (1 RTT):**
```
Client                                                      Server
  │── ClientHello: versions, cipher suites, random,          │
  │   key_share (my ECDHE public value), SNI = "amazon.in" ─►│
  │                                                          │ picks suite, computes shared secret
  │◄── ServerHello: key_share (server ECDHE public value) ───│
  │    {EncryptedExtensions, Certificate, CertificateVerify  │   ← already encrypted
  │     (signature proving it owns the cert's private key),  │
  │     Finished}                                            │
  │ verify cert chain + signature; compute same secret       │
  │── {Finished} + {HTTP request} ──────────────────────────►│   ← app data after 1 RTT
```
- **SNI** (Server Name Indication) tells the server which site's certificate to present, since one IP hosts many sites. It's visible to observers unless Encrypted ClientHello is used.
- **TLS 1.2** needs **2 RTTs** (a separate key exchange round). TLS 1.3 also removed weak algorithms and static-RSA key exchange.
- **0-RTT resumption:** a returning client sends encrypted data in its **first** flight, using a key from the previous session. ⚠ **Replayable:** an attacker can resend that data, so allow it only for idempotent requests.
- **HTTPS** = HTTP inside TLS on port 443. **mTLS** (mutual TLS) means the client also presents a certificate; it's used between microservices (service meshes) and in finance APIs.
- **Cost:** asymmetric crypto per handshake (offloaded to load balancers or terminated at the CDN), plus 1 RTT. Session resumption and connection reuse amortise both.

### 2.7 QUIC: transport + TLS over UDP (HTTP/3)
**QUIC** is a transport protocol implemented in **user space over UDP**. Google built it, and it's standardised as RFC 9000.
| Feature | Why it matters |
|---|---|
| Built-in **TLS 1.3**, combined with the transport handshake | new connection in **1 RTT** (TCP + TLS = 2), **0-RTT** on resumption |
| **Independent streams** | a lost packet stalls only the streams whose data it carried ⇒ no TCP-style head-of-line blocking |
| **Connection IDs** instead of the 5-tuple | a phone switching from Wi-Fi to cellular keeps the connection (**connection migration**) [APPLE] [QCOM] |
| In **user space** | new congestion control and features ship with app updates, not OS kernels |
| **Encrypted headers** (most transport fields) | middleboxes can't meddle ⇒ avoids protocol "ossification" |
| Better loss recovery | unique packet numbers (no retransmission ambiguity, so no need for Karn's rule), richer ACKs |
Costs: more CPU than kernel TCP (user-space crypto, no mature offloads yet); some networks block or throttle UDP, so clients fall back to TCP.

### 2.8 Other protocols to recognise
- **WebSocket:** starts as HTTP (`Upgrade: websocket` ⇒ 101), then becomes a full-duplex message channel over the same TCP connection. Used for chat, live dashboards, trading UIs. Alternatives: long polling, **Server-Sent Events** (one-way server push).
- **gRPC:** RPC over **HTTP/2** with **protobuf** binary messages. It supports streaming and is common between microservices [NVIDIA] [QCOM].
- **FIX** (Financial Information eXchange) [HFT]: a tag=value text protocol over **TCP**, used for orders and executions between brokers and exchanges. Exchanges also offer faster **binary** protocols (e.g. OUCH/ITCH-style formats).
- Email: SMTP (send, TCP 25/587), IMAP/POP3 (read). FTP: separate control and data connections (awkward with NAT). Name-only.

### 2.9 "What happens when you type www.google.com and press Enter?" (end to end)
Tie every note together. Say it **in layers and in time order**, and name the costs.
```
1. Browser parses the URL; checks HSTS list (force HTTPS); checks its cache for the page.
2. DNS (§2.2): browser cache → OS cache/hosts → recursive resolver → (root → .com → google.com
   authoritative) → A/AAAA record. To reach the resolver, the OS needs the GATEWAY's MAC:
   ARP (note 02) if not cached. Packets leave via the home router's NAT (note 04).
3. TCP 3-way handshake with the server's IP:443 (note 06) — 1 RTT.
   (Or QUIC over UDP: steps 3+4 merged.)
4. TLS 1.3 handshake (§2.6): certificate chain verified, keys derived — 1 RTT.
5. HTTP/2 GET / with Host, cookies, Accept headers (§2.3).
   Routing along the way: each router does longest-prefix match (note 05); MACs change per hop, IP stays.
   The packet likely reaches a nearby Google edge via ANYCAST/BGP (note 05).
6. Server side: L4 load balancer → L7 proxy/TLS terminator → web frontend → backend services
   (caches, databases, search index) → response (note 11).
7. Response travels back; TCP slow start + flow control shape delivery (note 07).
8. Browser parses HTML, discovers CSS/JS/images → more requests on the same multiplexed
   connection (or CDN hosts → more DNS/TCP/TLS); renders the page.
```
Interviewers then **drill into any step**: "How does the resolver find .com?", "What if the cert is invalid?", "Why is the second visit faster?" (DNS cache, TLS resumption, HTTP cache, warm connections).

### 2.10 Network attacks: a summary table
| Attack | How | Defence |
|---|---|---|
| **Man-in-the-middle** | intercept/modify traffic (via ARP spoofing, rogue Wi-Fi, DNS poisoning) | TLS with certificate validation, HSTS, certificate pinning (mobile apps) |
| ARP spoofing | forged ARP replies on the LAN (note 02) | Dynamic ARP Inspection, encryption |
| DNS cache poisoning | forged DNS replies | random port + ID, DNSSEC, DoH/DoT |
| SYN flood | half-open exhaustion (note 06) | SYN cookies, rate limits |
| **DDoS** (volumetric / amplification) | botnets, or reflection via DNS/NTP/memcached with spoofed source IPs (N4) | anycast absorption, scrubbing services (AWS Shield, Cloudflare), rate limiting, ingress filtering of spoofed IPs |
| Replay | resend captured messages | nonces/sequence numbers, timestamps; careful with 0-RTT |

### 2.11 How to read this note: must know / name-only / skip
| Must know | Name-only | Skip |
|---|---|---|
| DNS hierarchy, recursive vs iterative, caching/TTL, record types, why UDP; HTTP message format, methods + idempotency, status codes, statelessness + cookies; 1.1 vs 2 vs 3; caching/304; TLS goals, symmetric vs asymmetric, DH idea, certificates + chain, forward secrecy, TLS 1.3 1-RTT flow, 0-RTT replay; QUIC features; the full "type a URL" walkthrough; attacks table | DNSSEC, EDNS Client Subnet, HPACK/QPACK, Encrypted ClientHello, SMTP/IMAP/FTP, gRPC details | cipher-suite internals, X.509 field layouts, TLS record format, QUIC frame types |

---

## 3. Worked Numericals

**N1. DNS lookup time.** RTTs: client ↔ resolver 10 ms; resolver ↔ root 30 ms, ↔ .com TLD 40 ms, ↔ authoritative 50 ms.
- Cold cache: 10 + 30 + 40 + 50 = **130 ms**.
- Resolver has the .com and google.com NS records cached: 10 + 50 = 60 ms (authoritative only). Full answer cached: **10 ms**. Browser/OS cached: **~0 ms**.

**N2. Page with 10 embedded objects** (base HTML + 10 small images, same server). RTT = 50 ms; ignore transmission time and TLS.
| Scheme | RTTs | Time |
|---|---|---|
| HTTP/1.0 non-persistent, serial (handshake + request per object) | 2 × 11 = 22 | 1100 ms |
| HTTP/1.1 persistent, one request at a time | 2 (handshake + HTML) + 10 | 600 ms |
| Persistent with pipelining / HTTP/2 multiplexing | 2 + 1 | **150 ms** |
| HTTP/1.1 with 6 parallel connections | 2 + 2 (10 objects in 2 waves of ≤ 6; new connections' handshakes overlap) ≈ 4–5 | ~200–250 ms |

**N3. Connection setup cost (RTT 150 ms, e.g. Chennai → US East).** Time until the first response byte for a new connection:
| Stack | RTTs | Time |
|---|---|---|
| TCP + TLS 1.2 + request | 1 + 2 + 1 = 4 | 600 ms |
| TCP + TLS 1.3 + request | 1 + 1 + 1 = 3 | 450 ms |
| QUIC (HTTP/3) | 1 + 1 = 2 | 300 ms |
| QUIC 0-RTT resumption | 1 | **150 ms** |

**N4. DNS amplification.** An attacker sends 60-byte queries with the victim's spoofed IP to open resolvers; each answer is 3000 bytes.
- Amplification = 3000 / 60 = **50×**. With 100 Mbps of attack bandwidth ⇒ **5 Gbps** arriving at the victim, from thousands of "innocent" resolvers.

**N5. Diffie–Hellman with small numbers.** Public: prime p = 23, base g = 5. Client secret a = 6; server secret b = 15.
- Client sends A = gᵃ mod p = 5⁶ mod 23 = 15,625 mod 23 = **8**.
- Server sends B = gᵇ mod p = 5¹⁵ mod 23 = **19** (by squaring: 5² = 2, 5⁴ = 4, 5⁸ = 16 mod 23 ⇒ 16 · 4 · 2 · 5 = 640 mod 23 = 19).
- Client computes Bᵃ = 19⁶ mod 23 = **2**. Server computes Aᵇ = 8¹⁵ mod 23 = **2**. **Shared secret = 2.**
- An eavesdropper sees 23, 5, 8 and 19, but must solve the discrete logarithm to find a or b. That's easy here, infeasible for 2048-bit p or 256-bit elliptic curves.

**N6. 304 vs 200.** A 400 KB stylesheet, RTT 60 ms, 20 Mbps link, already in cache but stale.
- Full 200 response: 1 RTT + 400 × 8 × 1024 / 20 × 10⁶ ≈ 60 + 164 = **~224 ms** (plus slow-start rounds).
- Conditional 304: **~60 ms** (1 RTT, no body). With `max-age` still valid: **0 ms**, no request at all.

**Try it:** (a) RTT 80 ms; new HTTPS connection with TCP + TLS 1.3; DNS not cached (resolver path costs 120 ms). Time to the first response byte? (b) Same, but HTTP/3 with the DNS answer cached in the OS?
<details><summary>Answer</summary>(a) 120 (DNS) + 80 (TCP) + 80 (TLS) + 80 (request) = <b>360 ms</b>. (b) 0 + 80 (QUIC + TLS) + 80 (request) = <b>160 ms</b>.</details>

---

## 4. Interview Questions

### Basic
- **Q: What is DNS and how does resolution work?** Name → IP via a hierarchy: browser/OS cache → recursive resolver → root → TLD → authoritative, with each answer cached for its TTL (§2.2).
- **Q: Why does DNS use UDP?** Tiny request/response, no handshake (saves an RTT), and stateless servers handle huge query rates. TCP is used for large answers or zone transfers.
- **Q: GET vs POST? PUT vs POST?** GET reads (safe, idempotent, cacheable). POST creates or acts (not idempotent). PUT replaces at a known URL (idempotent, so it's safe to retry).
- **Q: 401 vs 403? 502 vs 504?** 401 = not authenticated; 403 = authenticated but forbidden. 502 = the upstream gave a bad/invalid response; 504 = the upstream timed out.

### Intermediate
- **Q: Recursive vs iterative DNS queries?** The stub asks the resolver recursively (final answer). The resolver queries root/TLD/authoritative iteratively (referrals).
- **Q: HTTP/1.1 vs HTTP/2 vs HTTP/3?** §2.4 table: persistent but serial vs multiplexed streams on one TCP connection (TCP HOL remains) vs QUIC streams over UDP (no transport HOL, 1-RTT setup, migration).
- **Q: HTTP is stateless. How do logins work?** Cookies carrying a session ID, or bearer tokens (JWT) sent with every request.
- **Q: Walk me through the TLS 1.3 handshake. How is the certificate verified?** §2.6. ClientHello + key share → ServerHello + key share + encrypted cert + signature + Finished → client Finished + data. Verification: the chain up to a trusted root, the signatures, the hostname (SAN), expiry/revocation, and CertificateVerify proving the server owns the private key.
- **Q: What is forward secrecy?** Ephemeral DH keys per session, so a later private-key compromise can't decrypt recorded past traffic.
- **Q: Why is the second visit to a site faster?** DNS cached, TLS session resumed (or 0-RTT), connection reused, HTTP cache/304, warm CDN edge.

### Advanced / killer follow-ups
- **Q [AMZN]: What happens when you type google.com and press Enter?** §2.9, in order: HSTS/cache → DNS (with ARP for the gateway, NAT) → TCP → TLS → HTTP request → routing/anycast → LB → backend → response, slow start → render + sub-resources. Expect drill-downs into any step.
- **Q: Why did HTTP/3 move to UDP instead of fixing TCP?** TCP lives in OS kernels and middleboxes and is ossified (hard to change or deploy). Its in-order byte stream inherently causes HOL blocking. QUIC in user space over UDP can evolve fast and multiplex independent streams.
- **Q: What's the risk of TLS 0-RTT?** The early data can be replayed by an attacker. Accept it only for idempotent requests, or use anti-replay mechanisms.
- **Q [AMZN]: How would you route users to the nearest region? DNS-based or anycast?** DNS (Route 53 latency/geo routing): flexible per-user answers, but bounded by TTL caching and resolver location. Anycast (BGP): instant failover and absorbs DDoS, but you have less control, and route changes can move existing flows. Large services combine both.
- **Q [APPLE]: How does a video call survive switching from Wi-Fi to LTE?** QUIC connection IDs (connection migration) or app-level reconnection; TCP would break because its 5-tuple changes.
- **Q: How do you defend a public API against DDoS?** Anycast + CDN/scrubbing (absorb volume), SYN cookies, rate limiting (429), autoscaling, caching, and filtering of amplification reflectors.
- **Q [HFT]: Why not use HTTP/REST for order entry?** Text parsing, headers, request/response framing and TLS overhead cost µs–ms. Trading uses persistent TCP sessions with compact binary or FIX messages (or kernel bypass), with no per-request setup.

---

## 5. Common Traps
- Saying DNS always uses UDP (large responses and zone transfers use TCP; DoH uses HTTPS).
- Confusing recursive (stub → resolver) with iterative (resolver → hierarchy).
- Thinking HTTP/2 removed all head-of-line blocking (TCP-level HOL remains).
- 401 vs 403 swapped; calling POST idempotent.
- Forgetting that TLS uses asymmetric crypto only to **agree on a key**; the bulk data is symmetric.
- Thinking HTTPS hides the domain from the network (SNI and DNS leak it unless ECH/DoH are used).
- Ignoring caching (DNS TTL, HTTP cache, TLS resumption) when explaining performance.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| DNS | root → TLD → authoritative; stub recursive, resolver iterative; TTL caching; UDP 53 (TCP for big) |
| Records | A, AAAA, CNAME, MX, NS, TXT, PTR, SOA, SRV |
| DNS routing | geo/latency/weighted/failover answers; bounded by TTL; poisoning + amplification attacks |
| HTTP | request line + headers + blank line + body; stateless; cookies/tokens |
| Methods | GET/HEAD safe; PUT/DELETE idempotent; POST neither |
| Status | 200/201/204 · 301/302/304 · 400/401/403/404/429 · 500/502/503/504 |
| Versions | 1.0 per request · 1.1 keep-alive · 2 multiplexed binary + HPACK (TCP HOL) · 3 QUIC/UDP |
| Caching | Cache-Control max-age; ETag + If-None-Match ⇒ 304 |
| TLS | confidentiality/integrity/authentication; ECDHE key share + certificate chain + symmetric bulk |
| TLS 1.3 | 1 RTT (1.2 = 2); 0-RTT resumption replayable; forward secrecy |
| QUIC | UDP + TLS 1.3; 1-RTT/0-RTT; independent streams; connection IDs (migration); user space |
| URL walkthrough | cache/HSTS → DNS (+ARP, NAT) → TCP → TLS → HTTP → LB → backend → slow start → render |
| Setup cost | TCP + TLS 1.3 + req = 3 RTT; QUIC = 2; 0-RTT = 1 |
