# 11. Data-Centre & GPU-Cluster Networking

> **Asked at:** [AMZN] L4 vs L7 load balancers, load-balancing algorithms, health checks, anycast, CDNs, VPC basics, "how does a request reach your service" · [NVIDIA] RDMA, InfiniBand vs RoCE, NVLink/NVSwitch, GPUDirect, all-reduce communication cost, lossless Ethernet (PFC/ECN), fat-tree fabrics · [DESHAW] load balancing + consistent hashing, tail latency · [QCOM] [APPLE] CDN/edge behaviour for apps (awareness).
> **Prerequisites:** [04](04-Network-Layer-IP-Subnetting-NAT-IPv6.md) (NAT, VXLAN mention in [02](02-Data-Link-Layer-and-Ethernet.md)), [05](05-Routing.md) (BGP, anycast, ECMP), [06](06-Transport-Layer-UDP-and-TCP-Connections.md)–[07](07-TCP-Reliability-Flow-and-Congestion-Control.md) (TCP, incast, ECN), [08](08-Application-Layer-DNS-HTTP-TLS-QUIC.md) (DNS routing, TLS, HTTP), [09](09-Sockets-and-the-Linux-Network-Stack.md) (copies and per-packet CPU cost). Also [COA 12 GPU Architecture](../Computer%20Organization%20and%20Architecture/12-GPU-Architecture.md), [OS 15 Virtualization](../Operating%20Systems/15-Virtualization-and-Containers.md) (SR-IOV, Nitro).

---

## 1. Intuition
- A big website is a **restaurant chain**. DNS and anycast send each customer to a nearby branch (**CDN/edge**). At the door, a host seats customers at free tables (the **load balancer**). Kitchens inside talk constantly over a well-planned corridor system (the **data-centre fabric**).
- A **GPU training cluster** is a kitchen where thousands of cooks must **combine their partial results after every step** (all-reduce). The corridors must be extremely fast and must never lose a plate, because one late cook makes everyone wait. So it uses special fabrics (**NVLink**, **InfiniBand/RoCE**) and lets the network card place data directly into another machine's memory (**RDMA**).

---

## 2. Deep Dive

### 2.1 Load balancers: L4 vs L7
A **load balancer (LB)** spreads client requests over many backend servers. It provides scale, availability (unhealthy servers are removed) and a single public address.
| | **L4 (transport-level)** | **L7 (application-level)** |
|---|---|---|
| Looks at | IPs + ports (the 5-tuple) only | full HTTP: URL path, headers, cookies, method |
| Connection handling | forwards packets/connections. The client's TCP connection goes **through** to a backend (or is NAT-ed) | **terminates** the client's TCP + TLS connection and opens its own connection to the backend (a *proxy*) |
| Can do | per-connection balancing, very high throughput, low latency | path-based routing (`/api` → service A), header/cookie stickiness, retries, rate limiting, auth, compression, caching, gRPC/HTTP/2 per-request balancing |
| Cost | cheap per packet; millions of connections | TLS + HTTP parsing ⇒ more CPU, a bit more latency |
| Examples | AWS **NLB**, Google Maglev, Facebook Katran (XDP), LVS, HAProxy in TCP mode | AWS **ALB**, nginx, Envoy, HAProxy in HTTP mode, API gateways |
- **Direct Server Return (DSR):** an L4 LB forwards requests, but backends reply **directly** to clients (responses are much larger than requests), so the LB isn't the bottleneck.
- **Health checks:** periodic TCP connects or HTTP `GET /health`; failing backends are removed. *Trap:* a health check that's too shallow (the process is up but the database is down) or too deep (a dependency blip removes every backend at once).
- **Connection draining:** when removing a backend, stop sending it **new** requests but let in-flight ones finish.
- **TLS termination** at the LB centralises certificates and offloads crypto from backends (re-encrypt to the backends if required).
- **L4 + L7 tiers together:** anycast/BGP → L4 LB fleet (ECMP across LB nodes) → L7 proxy fleet → services. This is how large services front their traffic.

**Balancing algorithms:**
| Algorithm | How | Note |
|---|---|---|
| Round robin / weighted RR | rotate through backends | ignores current load |
| **Least connections / least outstanding requests** | pick the least busy | adapts to slow requests |
| **Power of two choices** | pick 2 random backends, send to the less loaded one | nearly as good as global least-loaded, with no global state (Envoy, Nginx) |
| Hash (source IP / header / 5-tuple) | same client → same backend (stickiness) | uneven if keys are skewed |
| **Consistent hashing** (incl. Maglev hashing) | adding or removing one backend remaps only ~1/N of the keys | caches, sharded state, L4 LBs that must keep flows on the same backend |
**Consistent hashing in one paragraph:** place servers and keys on a hash ring (0 … 2³² − 1). Each key goes to the **next server clockwise**. Removing a server moves only *its* keys to the next server. **Virtual nodes** (each server at many ring points) even out the load. With plain `hash mod N`, changing N remaps almost **every** key (N3).

### 2.2 Getting users to the right place: DNS, anycast, CDN
- **DNS-based** routing (note 08 §2.2): latency/geo/weighted answers. Simple, but bounded by TTL caching.
- **Anycast** (note 05 §2.6): the same IP is announced by BGP from many sites, so users reach the topologically nearest site. It gives fast failover (withdraw the route) and spreads DDoS traffic over all sites. Used for DNS, CDNs and L4 LB front doors (AWS Global Accelerator, Cloudflare).
- **CDN** (Content Delivery Network: CloudFront, Akamai, Cloudflare): edge servers in hundreds of cities **cache** static content and **terminate TCP/TLS close to the user**. That cuts the RTT that dominates small transfers (note 07 N2). Dynamic requests travel from the edge to the origin over warm, long-lived, tuned connections. Cache keys, TTLs and invalidation are the hard parts.

### 2.3 Inside the data centre: Clos / leaf–spine fabrics
Old data-centre networks were **trees** (access → aggregation → core), whose upper links were oversubscribed bottlenecks. Modern ones are **Clos (leaf–spine / fat-tree)** fabrics:
```
          Spine 1        Spine 2        Spine 3        Spine 4
           │ ╲  ╲  ╱  ╱ │ ╲   ╱  ╱ │   ╲  ╱  ╱  │        (every leaf connects to every spine)
   ┌───────┴──┐  ┌──────┴───┐  ┌─────┴────┐  ┌──┴───────┐
   │  Leaf 1  │  │  Leaf 2  │  │  Leaf 3  │  │  Leaf 4  │   ← "top-of-rack" (ToR) switches
   └┬───┬───┬─┘  └┬───┬───┬─┘  └┬───┬───┬─┘  └┬───┬───┬─┘
    servers        servers        servers        servers
```
- Any two servers on different leaves are exactly **leaf → spine → leaf** apart (3 switch hops). There are **many equal-cost paths** (one per spine), used with **ECMP** flow hashing (note 05 §2.7).
- **Oversubscription ratio** = server-facing bandwidth ÷ uplink bandwidth on a leaf. 1:1 = **non-blocking** (full bisection bandwidth); 3:1 is common for general services. AI clusters aim for 1:1 (N4).
- **East–west** traffic (server ↔ server) dominates over **north–south** (in/out of the data centre). Microservices and distributed training are all east–west.
- **Cloud virtual networks** (an AWS VPC) run as an **overlay** on top of this physical fabric: tenant packets are encapsulated (VXLAN-like or proprietary, e.g. AWS's on **Nitro** cards), so tenants get private address spaces, security groups and routing tables, independent of the physical network.

### 2.4 Why ordinary TCP isn't enough for HPC and AI clusters
Distributed training and HPC move **huge, synchronised** data volumes (gradients of billions of parameters every step), and every GPU waits for the slowest transfer. Kernel TCP hurts here (notes 07, 09):
- **CPU cost:** copies (user ↔ kernel), per-packet stack processing and interrupts. At 400 Gbps that's many cores (note 09 N1, N4).
- **Latency:** ~5–10 µs+ through the kernel stack, plus jitter.
- **Loss recovery:** drops cause timeouts (≥ 200 ms) and **incast** collapses (note 07 §2.6), and one slow flow stalls the whole collective operation.

### 2.5 RDMA: Remote Direct Memory Access
**RDMA** lets an application **read or write another machine's memory directly through the NICs**: **no remote CPU involvement, no kernel on the data path, no copies**.
```
 Normal TCP:   app buf ─copy→ kernel ─stack→ NIC ═══ NIC ─stack→ kernel ─copy→ app buf (+ remote CPU wakes up)
 RDMA WRITE:   app buf ──DMA──► NIC ═══════════════ NIC ──DMA──► remote app buf   (remote CPU not involved)
```
**Key ideas (and terms):**
- **Kernel bypass:** the app talks to the NIC through **queue pairs (QPs)**. A QP is a send queue + a receive queue of work requests in user memory. The app posts work and rings a doorbell, and later polls a **completion queue (CQ)** for results.
- **Memory registration:** before use, a buffer is **registered**. The kernel **pins** its pages (they can't be swapped or moved, because the NIC uses physical addresses; [OS 09 §2.7](../Operating%20Systems/09-Virtual-Memory-Page-Faults-and-Replacement.md)) and gives the NIC the translation. The remote side needs a **key** (rkey) to access it, which provides protection.
- **Verbs** (the API, `libibverbs`):
  - **one-sided** RDMA **READ/WRITE**: the initiator names the remote address, and the remote CPU never sees it;
  - **two-sided** SEND/RECV: the receiver pre-posts buffers;
  - **atomics** (fetch-and-add, compare-and-swap on remote memory).
- **Transport in the NIC:** reliability (sequence numbers, ACKs, retransmission) is implemented **in NIC hardware**, so there's no OS stack.
- **Result:** ~**1–2 µs** latency and full line rate (100–400 Gbps per port), with nearly zero CPU. Used for distributed training (NCCL), HPC (MPI), distributed storage and databases, and some trading back-ends.

**Three ways to carry RDMA:**
| | **InfiniBand (IB)** | **RoCE v2** (RDMA over Converged Ethernet) | **iWARP** |
|---|---|---|---|
| Network | its own L1–L4 stack, own switches, subnet manager | RDMA transport inside **UDP/IP over Ethernet** (routable) | RDMA over TCP |
| Loss handling | **credit-based link-level flow control ⇒ lossless by design** | needs a **lossless** or near-lossless Ethernet: **PFC** + **ECN/DCQCN** congestion control (or newer loss-tolerant designs) | TCP handles loss |
| Pros | lowest latency, mature for HPC/AI (NVIDIA Quantum switches) | reuses Ethernet ops, tools and switches (NVIDIA Spectrum-X, cloud fabrics) | works on any network |
| Cons | separate fabric, specialised skills | PFC side-effects: head-of-line blocking, pause storms, deadlocks; careful tuning | higher latency, less common |
- **PFC** (Priority Flow Control): when a switch buffer fills, it sends **PAUSE** frames upstream for one traffic class, so no packet is dropped. Pauses can spread backwards and block unrelated flows (congestion spreading). Hence ECN-based congestion control (**DCQCN**) is added to keep queues short, so PFC rarely triggers.
- Newer AI Ethernet designs (e.g. the Ultra Ethernet effort; packet spraying with out-of-order delivery) aim to tolerate loss instead of depending on PFC. Name-level.

### 2.6 GPU interconnects: NVLink, NVSwitch, GPUDirect [NVIDIA]
**Inside a server ("scale-up"):**
- **PCIe** links GPUs to CPUs and NICs: Gen5 x16 ≈ **64 GB/s per direction**. That's too slow for GPU-to-GPU traffic in big models.
- **NVLink:** NVIDIA's GPU-to-GPU link. H100 has **18 NVLink 4 links ≈ 900 GB/s** total bandwidth per GPU (~14× PCIe Gen5 x16).
- **NVSwitch:** a switch chip connecting all GPUs in a server (e.g. 8 GPUs in a DGX/HGX H100) **all-to-all at full NVLink bandwidth**. The NVL72 rack systems (GB200) extend one NVLink domain to 72 GPUs, so they behave like one giant GPU for communication.

**Between servers ("scale-out"):**
- InfiniBand or RoCE NICs (ConnectX, 400–800 Gbps), typically **one NIC per GPU**, in a non-blocking fat-tree.
- **GPUDirect RDMA:** the NIC DMAs **directly to and from GPU memory** over PCIe, skipping the bounce through host RAM and the CPU. **GPUDirect Storage:** NVMe/storage ↔ GPU memory directly.

```
   Server A                                         Server B
 GPU0 ═NVLink/NVSwitch═ GPU1 … GPU7               GPU0 … GPU7
   │ PCIe (GPUDirect RDMA)                          │
  NIC0 ─────────── InfiniBand / RoCE fabric ─────── NIC0
```

### 2.7 Collective communication: all-reduce
**Data-parallel training:** every GPU computes gradients on its own slice of the batch. Then **all GPUs must end up with the sum (or average) of all gradients** before the next step. This operation is **all-reduce**. (Other collectives: broadcast, reduce, all-gather, reduce-scatter, all-to-all.) **NCCL** (NVIDIA Collective Communications Library) implements them over NVLink, PCIe, IB and RoCE.

**Ring all-reduce** (bandwidth-optimal):
- Arrange N GPUs in a logical ring and split the data (size S) into N chunks.
- **Phase 1, reduce-scatter:** N − 1 steps. Each GPU sends one chunk to its neighbour and adds the chunk it receives. Afterwards, each GPU holds the **full sum of one chunk**.
- **Phase 2, all-gather:** N − 1 steps passing the completed chunks around. Afterwards, everyone has every summed chunk.
- **Data each GPU sends** = 2 × (N − 1)/N × S ≈ **2S**, independent of N. That's why it scales in bandwidth.
- **Time** ≈ 2(N − 1) × α + 2 × (N − 1)/N × S / B, with α = per-step latency and B = per-link bandwidth. The α term grows with N, so huge clusters use **tree** or **hierarchical** algorithms: reduce inside a node over NVLink first, then across nodes over IB, then broadcast back inside each node.
- **Why the network design follows from this:** every step waits for the **slowest link**. Congestion, a hash collision (note 05 N5) or a single retransmission timeout stalls the entire cluster ⇒ non-blocking fabrics, adaptive routing, lossless or loss-tolerant transports, and topology-aware placement.

### 2.8 How to read this note: must know / name-only / skip
| Must know | Name-only | Skip |
|---|---|---|
| L4 vs L7 table; LB algorithms + consistent hashing; health checks, draining, TLS termination; DNS vs anycast vs CDN; leaf–spine + ECMP + oversubscription; why TCP falls short for AI/HPC; RDMA idea (bypass, zero-copy, one-sided, pinned registered memory, QPs/CQs); IB vs RoCE + why RoCE needs lossless (PFC/ECN); NVLink/NVSwitch vs PCIe; GPUDirect RDMA; ring all-reduce traffic 2(N−1)/N·S and its time formula | DSR, Maglev, VPC overlays, DCQCN, Ultra Ethernet, NVL72, SHARP in-network reduction, iWARP | verbs API details, IB subnet-manager internals, NCCL tuning knobs |

---

## 3. Worked Numericals

**N1. L7 LB capacity.** Each TLS 1.3 handshake costs 1 ms of CPU (ECDHE + signature) on an LB node with 32 cores. Peak: 50,000 **new** connections/s.
- CPU needed for handshakes = 50,000 × 1 ms = **50 core-seconds per second ⇒ 50 cores** ⇒ at least **2 nodes** (before HTTP work).
- With session resumption for 80 % of connections (~0.1 ms each): 10,000 × 1 + 40,000 × 0.1 = 14,000 ms ⇒ **14 cores**. Connection reuse and resumption matter enormously.

**N2. Power of two choices (intuition).** 100 backends, 100 requests placed at random: the most loaded backend gets about **4–5** requests (≈ ln n / ln ln n).
- With "pick 2, choose the less loaded", the max drops to about **2–3** (≈ ln ln n / ln 2). That's an exponential improvement from one extra sample, with no global coordination.

**N3. Consistent hashing vs mod N.** 1,000,000 cached keys on 10 servers; one server is added.
- `hash mod N`: a key stays only if hash mod 10 = hash mod 11 ⇒ only ~1/11 of keys stay ⇒ **~909,000 keys move** (a cache-miss storm).
- Consistent hashing: the new server takes ~1/11 of the ring ⇒ **~91,000 keys move**, all of them onto the new server.

**N4. Leaf oversubscription.** A leaf has 48 server ports at 25 Gbps and 6 uplinks at 100 Gbps.
- Downlink = 48 × 25 = 1,200 Gbps; uplink = 600 Gbps ⇒ **2:1** oversubscription. If all servers send off-rack at once, each gets ≤ 12.5 Gbps.
- For an AI pod that needs non-blocking: 12 × 100 Gbps uplinks (or 48 servers' worth of 400G uplinks to match 400G NICs).

**N5. Ring all-reduce time.** N = 8 GPUs, gradients S = 10 GB (FP16 ≈ 5 B parameters), per-link bandwidth B = 50 GB/s (400 Gbps NIC), per-step latency α = 10 µs.
- Bytes sent per GPU = 2 × (7/8) × 10 GB = **17.5 GB** ⇒ bandwidth time = 17.5 / 50 = **0.35 s**.
- Latency term = 2 × 7 × 10 µs = 0.14 ms (negligible here).
- If the same 8 GPUs are in one NVLink domain at ~450 GB/s per direction: 17.5 / 450 ≈ **39 ms**. That's why the reduction happens over NVLink inside a node first.
- With N = 1024 GPUs: bandwidth time ≈ 2 × S / B (the same), but the latency term = 2 × 1023 × 10 µs ≈ **20 ms** per all-reduce ⇒ tree/hierarchical algorithms.

**N6. RDMA vs TCP CPU cost.** Receive 200 Gbps (25 GB/s).
- TCP: copies alone at ~10 GB/s per core ⇒ 2.5 cores, plus per-packet stack processing (~2 M full-size packets/s arrive; GRO merges them, but the work is still large) ⇒ realistically **several more cores**.
- RDMA WRITE: the NIC places the data directly; the CPU only polls completions ⇒ **≈ 0 cores** on the receiver, and the GPUs or app get those cores back.

**Try it:** (a) 16 GPUs run a ring all-reduce of 4 GB at 25 GB/s per link with α = 5 µs. Estimate the bandwidth and latency terms. (b) 20 cache servers become 21 with consistent hashing over 2 M keys. Roughly how many keys move?
<details><summary>Answer</summary>(a) Bytes = 2 × 15/16 × 4 = 7.5 GB ⇒ 7.5 / 25 = <b>0.3 s</b>; latency = 2 × 15 × 5 µs = <b>0.15 ms</b>. (b) ≈ 2 M / 21 ≈ <b>95,000</b> keys (vs ~1.9 M with mod N).</details>

---

## 4. Interview Questions

### Basic
- **Q: L4 vs L7 load balancer?** §2.1 table. L4 forwards by 5-tuple (fast, protocol-agnostic); L7 terminates TCP/TLS and routes by HTTP content (smart, costlier).
- **Q: What does a CDN do?** It caches content at edges near users and terminates connections close to them ⇒ lower RTT and less origin load; reached via DNS/anycast.
- **Q: What is RDMA?** NIC-to-NIC direct memory access: kernel bypass, zero copy, no remote CPU, with reliability in NIC hardware ⇒ ~µs latency at line rate.

### Intermediate
- **Q: Which LB algorithm would you choose and why?** Least-outstanding-requests or power-of-two choices for uneven request costs; consistent hashing when backends hold per-key state (caches); weighted round robin for heterogeneous capacity.
- **Q: Explain consistent hashing.** Servers and keys on a ring; a key goes to the next server clockwise; adding or removing a server moves ~1/N of the keys; virtual nodes balance the load (N3).
- **Q: What is a leaf–spine fabric and why ECMP?** Every leaf connects to every spine ⇒ a uniform 3-hop path and many equal-cost paths; ECMP hashes flows across them for bisection bandwidth.
- **Q: Anycast vs DNS-based global load balancing?** Anycast: BGP picks the nearest site; fast failover, DDoS absorption, less control. DNS: per-user answers and weights, but bounded by TTL caching and resolver location.
- **Q [NVIDIA]: InfiniBand vs RoCE?** IB: native lossless (credit-based) fabric, lowest latency, separate ecosystem. RoCE v2: RDMA over UDP/IP Ethernet; needs PFC + ECN (DCQCN) tuning, but reuses Ethernet.

### Advanced / killer follow-ups
- **Q [NVIDIA]: Why does RoCE need a lossless network, and what problems does PFC cause?** The NIC's simple go-back-N-style recovery collapses under loss. PFC pauses prevent drops but cause head-of-line blocking, congestion spreading, pause storms and possible deadlocks ⇒ add ECN-based congestion control so PFC rarely fires.
- **Q [NVIDIA]: Derive the communication cost of ring all-reduce. When is it not optimal?** Each GPU sends 2(N−1)/N·S; time ≈ 2(N−1)α + 2(N−1)S/(NB). It's bandwidth-optimal, but the latency term grows linearly in N ⇒ trees, hierarchical (NVLink inside the node first), or in-network reduction (SHARP) for large N or small messages.
- **Q [NVIDIA]: What is GPUDirect RDMA and why does it matter?** The NIC DMAs straight to and from GPU memory over PCIe ⇒ no host-memory bounce or CPU copy ⇒ lower latency, more bandwidth, free CPU.
- **Q [NVIDIA]: NVLink vs PCIe vs InfiniBand: where is each used?** NVLink/NVSwitch: GPU↔GPU inside a node or rack (~900 GB/s per GPU). PCIe: GPU↔CPU/NIC/storage (~64 GB/s Gen5 x16). IB/RoCE: across nodes (400–800 Gbps per NIC).
- **Q [AMZN]: Trace a request from a user to a service behind ALB in a VPC.** DNS (Route 53) → nearest edge/anycast (optional CloudFront) → ALB nodes (TLS termination, path routing) → target group instances/containers in private subnets over the VPC overlay (security groups) → response back (optionally through NAT for backend egress).
- **Q: Your health checks pass but users see errors. Why?** The check is too shallow (process alive, dependencies dead) or checks the wrong port/path; or the errors come from a dependency/LB timeout (502/504). Use meaningful readiness checks, but avoid cascading removals when a shared dependency blips.
- **Q: How would you keep tail latency low in a scatter-gather service (one request fans out to 100 servers)?** The p99 of the whole request ≈ the worst of 100 ⇒ hedged requests (send a duplicate after the p95 delay), tied requests, partial results, less fan-in (avoid incast), and ECN/DCTCP.

---

## 5. Common Traps
- Saying an L4 LB can route by URL (it can't see HTTP).
- Forgetting that L7 LBs terminate TLS (certificates and CPU live there).
- Using `hash mod N` for caches that resize (a mass remap).
- Assuming RDMA needs no network support (RoCE needs lossless/ECN tuning; IB has its own fabric).
- Forgetting that RDMA buffers must be registered (pinned): registration is expensive, so do it once and reuse.
- Thinking ring all-reduce traffic grows with N per GPU (it's ≈ 2S; the latency term grows).
- Confusing NVLink (inside a node/rack) with InfiniBand (between nodes).

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| L4 vs L7 | 5-tuple forwarding (NLB, Maglev, Katran) vs terminate TCP/TLS + HTTP routing (ALB, Envoy, nginx) |
| LB extras | health checks (not too shallow or deep), draining, TLS termination, DSR |
| Algorithms | RR/weighted · least connections · power of two choices · consistent hashing (~1/N moves) |
| Global | DNS routing (TTL-bound) · anycast (BGP nearest, DDoS) · CDN (cache + nearby TLS) |
| Fabric | leaf–spine Clos, 3 hops, ECMP, oversubscription (1:1 for AI); east–west dominant; VPC overlay |
| TCP limits | copies + per-packet CPU, kernel latency, loss ⇒ RTO/incast stalls collectives |
| RDMA | QPs/CQs in user space, registered pinned memory + rkey, one-sided READ/WRITE, NIC-level reliability, ~1–2 µs |
| IB vs RoCE | credit-based lossless own fabric vs RDMA in UDP/IP needing PFC + ECN (DCQCN) |
| GPU links | NVLink 4 ~900 GB/s per H100 + NVSwitch all-to-all; PCIe Gen5 x16 ~64 GB/s; GPUDirect RDMA |
| All-reduce | ring: reduce-scatter + all-gather; per GPU 2(N−1)/N·S; T ≈ 2(N−1)α + 2(N−1)S/(NB); hierarchical for big N |
