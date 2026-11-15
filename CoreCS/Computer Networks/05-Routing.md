# 05. Routing

> **Asked at:** [AMZN] [DESHAW] forwarding vs routing, longest-prefix match (and coding it with a trie), distance vector vs link state, count-to-infinity, OSPF vs BGP, how traffic reaches a cloud region · [NVIDIA] ECMP and flow hashing in GPU-cluster fabrics · [HFT] deterministic paths, why per-packet load balancing reorders packets · [QCOM] [TI] router control plane vs data plane (embedded networking software).
> **Prerequisites:** [04 Network Layer](04-Network-Layer-IP-Subnetting-NAT-IPv6.md) (prefixes, CIDR, aggregation, TTL).

---

## 1. Intuition
- **Forwarding** is what a driver does at each junction: look at the destination on the signboard and turn. It's a quick, local decision.
- **Routing** is how the signboards get written in the first place. Either:
  - **distance vector:** each junction asks its neighbours "how far is Mumbai from you?";
  - **link state:** every junction gets the whole map and computes the shortest paths itself.

Between countries (different organisations), the choice is about **business and policy**, not the shortest road. That's BGP.

---

## 2. Deep Dive

### 2.1 Forwarding: the table and longest-prefix match
Each router has a **forwarding table**: a list of (**prefix → next hop / output port**). For every packet it finds the entry matching the destination IP. Usually several prefixes match, and the rule is:

> **Longest-prefix match (LPM):** among all matching prefixes, use the **most specific** one (largest /p).
```
Forwarding table                     Packet dst        Matches              Chosen
10.0.0.0/8      → port A             10.1.2.5          /8, /16, /24, /0     10.1.2.0/24 → C
10.1.0.0/16     → port B             10.1.3.4          /8, /16, /0          10.1.0.0/16 → B
10.1.2.0/24     → port C             10.200.1.1        /8, /0               10.0.0.0/8  → A
0.0.0.0/0       → port D (default)   8.8.8.8           /0 only              default     → D
```
- **Why longest:** CIDR lets a big block be aggregated (/8) while an exception inside it (/24) goes elsewhere. The more specific route is more precise information.
- **0.0.0.0/0 = the default route:** it matches everything, so it's used only when nothing else matches ("send to my ISP").
- **How it's done fast:**
  - **Software:** a **trie** (prefix tree) over the address bits (§4 has code). Linux uses a compressed trie (LC-trie).
  - **Hardware routers:** **TCAM** (ternary content-addressable memory). It compares the address against **all entries in parallel in one cycle**, with "don't care" bits for the host part, and returns the highest-priority (longest) match. A core router must do this for ~100 million packets per second per port.

### 2.2 Routing: the big picture
- Model the network as a **graph**: routers are nodes, links are edges, and each link has a **cost** (set by the operator: often 1 per hop, or inversely proportional to bandwidth). Goal: the **least-cost path** from every router to every destination.
- **Static routing:** an admin types the routes. Fine for small or edge networks (a home router's single default route).
- **Dynamic routing:** routers run a protocol, exchange information, and react to failures automatically.
- **Autonomous System (AS):** a network under one administration (an ISP, Amazon, Google, IITM), identified by an **AS number** (e.g. AS16509 = Amazon). Routing is split into two levels:
  - **Intra-domain** (inside one AS; the goal is performance): **OSPF**, **IS-IS** (link state), **RIP** (distance vector).
  - **Inter-domain** (between ASes; the goal is policy): **BGP**.

### 2.3 Distance-vector routing (Bellman-Ford): "ask your neighbours"
Each router x keeps a **distance vector** D_x = its current best-known cost to every destination. It knows only the costs of its **own links**, plus the vectors its **neighbours** send it.

**Bellman-Ford equation:**
```
D_x(y) = min over neighbours v of { c(x, v) + D_v(y) }
         (cost to reach neighbour v, plus v's claimed cost from v to y)
```
**Algorithm:** each router periodically (or when something changes) sends its vector to its neighbours. On receiving a vector, it recomputes with the equation; if its own vector changed, it tells its neighbours. This is **iterative, asynchronous and distributed**, and it converges to the true least costs (N3).

**The bad-news problem: count-to-infinity** (N4).
- Good news (a cost decreases) spreads fast.
- Bad news (a link breaks or gets expensive) spreads slowly: router y thinks "z can reach x cheaply", but z's cheap path actually goes **through y**. The two bounce increasing estimates back and forth, a few units per exchange, until reaching the true cost or "infinity".
- **Fixes:**
  - **Split horizon:** don't advertise a route back to the neighbour you learned it from.
  - **Poison reverse:** advertise it back as **∞**, so the neighbour won't route through you.
  - Both fix loops between **2** routers, not every longer loop.
  - **Maximum metric:** RIP (Routing Information Protocol, an old distance-vector protocol) treats **16 hops = infinity**, so counting stops and RIP networks are limited to 15 hops.

### 2.4 Link-state routing (Dijkstra): "everyone gets the full map"
1. Each router discovers its neighbours (OSPF sends **Hello** packets) and measures link costs.
2. It **floods** a **link-state advertisement (LSA)**, "I am R3; my links: R1 cost 2, R5 cost 1", to **every** router in the area. Sequence numbers make sure the newest LSA wins and flooding stops.
3. Every router now has the **same complete map** (the link-state database) and runs **Dijkstra's shortest-path algorithm** with itself as the source.
4. For each destination, it keeps only the **first hop** of the shortest path. That becomes its forwarding table.

**Dijkstra in one paragraph:** keep a set N′ of nodes whose shortest distance is final. Start with N′ = {source}, with tentative distances D(v) = c(source, v) for neighbours and ∞ for others. Repeatedly pick the node w ∉ N′ with the **smallest D(w)**, add it to N′, and **relax** its neighbours: D(v) = min(D(v), D(w) + c(w, v)). Cost O(n²) with an array, or O((n + E) log n) with a heap (N2 traces it).

- **OSPF** (Open Shortest Path First): the standard link-state protocol inside enterprises and ISPs. It splits big networks into **areas** (flooding stays inside an area; area 0 is the backbone) and converges quickly after failures (sub-second with tuning). **IS-IS** is similar and popular in large ISPs and data centres.
- *Name-only:* if link costs depend on current load, link-state routes can **oscillate** (everyone moves to the idle path, which becomes congested, so everyone moves back). That's why costs are usually static.

### 2.5 Distance vector vs link state
| | Distance vector (RIP, Bellman-Ford) | Link state (OSPF, IS-IS, Dijkstra) |
|---|---|---|
| Each router knows | only neighbours' distance vectors | the **whole topology** |
| Sends | its full vector, to **neighbours** | its own links (LSA), **flooded to all** |
| Computation | Bellman-Ford, distributed, iterative | Dijkstra, locally, on the full map |
| Convergence | slow; **count-to-infinity** on bad news | fast |
| Robustness | one wrong vector spreads everywhere ("routing by rumour") | each router computes its own table ⇒ errors are contained |
| Memory / CPU | low | higher (full map + Dijkstra) |
| Used | small/legacy (RIP); the *idea* lives on in BGP | most enterprise and ISP interiors |

### 2.6 BGP: routing between organisations
**BGP** (Border Gateway Protocol) is how the ~75,000+ ASes of the Internet tell each other which prefixes they can reach. It runs over **TCP port 179** between neighbouring routers.
- **Path vector:** an advertisement carries the prefix **plus the full list of ASes** to traverse, e.g. `203.0.113.0/24, AS-PATH [65002 65003 65010]`.
  - **Loop prevention:** a router rejects any path that already contains its own AS number. This avoids count-to-infinity.
- **Policy, not shortest path:** each AS chooses routes by **business relationships**:
  - **customers** pay you (prefer their routes; advertise them to everyone);
  - **peers** exchange traffic for free (advertise only your customers' routes to them);
  - **providers** you pay (use them as the last resort).
  - An AS won't carry traffic between two of its providers for free. Hence BGP routes are often **not** the shortest.
- **Route selection** (simplified order): highest **local preference** (the operator's policy) → **shortest AS path** → lowest origin type/MED → prefer eBGP over iBGP → lowest IGP cost to the exit ("hot-potato": hand traffic off as early as possible) → tie-breakers.
- **eBGP vs iBGP:** eBGP runs between routers of *different* ASes; iBGP spreads those external routes *inside* one AS.
- **Failures and attacks:**
  - **Route hijacks/leaks:** an AS wrongly announces someone else's prefix, often more specific, so LPM attracts the traffic. Examples: 2008 Pakistan Telecom took YouTube offline globally; route leaks have rerouted big cloud providers.
  - Defences: **RPKI** (cryptographically signed prefix ownership) and filtering.
  - The **2021 Facebook outage** was Facebook's own BGP withdrawing its routes: its DNS servers became unreachable worldwide.
- **Anycast:** many sites announce the **same prefix**, and BGP sends each user to the "nearest" one. Used by DNS root servers, CDNs and Google's 8.8.8.8 (note 11).

### 2.7 ECMP: using several equal paths at once
**ECMP** (Equal-Cost Multi-Path): if several next hops have the same cost, use all of them to spread load. Data-centre fabrics are built around this (note 11).
- **Per-flow hashing:** hash the **5-tuple** (src IP, dst IP, src port, dst port, protocol) and pick a path by `hash mod #paths`. All packets of one TCP connection take the **same** path.
- **Why not per packet?** Packets of a flow on different paths arrive **out of order**. TCP reads reordering as loss (duplicate ACKs ⇒ spurious fast retransmit ⇒ it shrinks its window, note 07). Throughput drops.
- **Downside:** two big ("elephant") flows can hash onto the same path and congest it while another path sits idle (N5). Remedies: flowlet switching, packet spraying with reorder-tolerant transports (used in some AI-cluster fabrics) [NVIDIA].

### 2.8 Inside a router (software view)
```
            ┌────────────── CONTROL PLANE (CPU, software) ───────────────┐
            │  OSPF / BGP processes → routing table (RIB) → best routes  │
            └──────────────────────────┬─────────────────────────────────┘
                                       │ installs into
            ┌──────────────────────────▼─── DATA PLANE (ASIC / NPU) ─────┐
 in port ──►│ parse → LPM lookup in FIB (TCAM) → queue at output port ───┼──► out port
            └────────────────────────────────────────────────────────────┘
```
- **RIB** (Routing Information Base): every route learned, with attributes, kept by the control-plane software. **FIB** (Forwarding Information Base): only the chosen best routes, programmed into the hardware.
- Output **queues/buffers** absorb bursts. Too-large buffers cause **bufferbloat**: huge queuing delays under load (note 07).
- **SDN** (Software-Defined Networking, name-level): a central controller computes routes and programs the switches' tables (e.g. OpenFlow; Google's B4 WAN), instead of each router running distributed protocols.

### 2.9 How to read this note: must know / name-only / skip
| Must know | Name-only | Skip |
|---|---|---|
| forwarding vs routing; LPM + default route + trie/TCAM; AS, intra vs inter-domain; Bellman-Ford equation, a DV trace, count-to-infinity + split horizon/poison reverse; Dijkstra trace; DV vs LS table; BGP = path vector + policy + loop prevention; ECMP per-flow hashing and why | OSPF areas, RIP 15-hop limit, BGP attribute order, RPKI, anycast details, SDN | OSPF packet types and LSA formats, BGP message formats, MPLS label switching internals |

---

## 3. Worked Numericals

**N1. Longest-prefix match in binary.** Forwarding table:
```
11001000 00010111 00010***  ********  → link 0      (200.23.16.0/21)
11001000 00010111 00011000  ********  → link 1      (200.23.24.0/24)
11001000 00010111 00011***  ********  → link 2      (200.23.24.0/21)
otherwise                              → link 3
```
- 200.23.22.161 = …00010110…: matches only /21 #1 ⇒ **link 0**.
- 200.23.24.170 = …00011000…: matches the /24 and /21 #2 ⇒ the longer is the /24 ⇒ **link 1**.
- 200.23.25.1 = …00011001…: matches /21 #2 only ⇒ **link 2**.
- 200.24.0.1 ⇒ no match ⇒ **link 3** (default).

**N2. Dijkstra from u.** Links: u–v 2, u–x 1, u–w 5, v–x 2, v–w 3, x–w 3, x–y 1, w–y 1, w–z 5, y–z 2.
| Step | N′ | D(v), p | D(w), p | D(x), p | D(y), p | D(z), p |
|---|---|---|---|---|---|---|
| 0 | u | 2, u | 5, u | **1, u** | ∞ | ∞ |
| 1 | u x | **2, u** | 4, x | | 2, x | ∞ |
| 2 | u x v | | 4, x | | **2, x** | ∞ |
| 3 | u x v y | | **3, y** | | | 4, y |
| 4 | u x v y w | | | | | **4, y** |
| 5 | all | | | | | |
(p = predecessor on the best path. At step 2, v and y tie at 2; either can go first.)
Shortest costs from u: v 2, x 1, y 2, w 3, z 4. **u's forwarding table:** v → via v; x, y, w, z → **via x** (e.g. z: u → x → y → z).

**N3. Distance vector converging.** Three routers: c(x, y) = 2, c(y, z) = 1, c(x, z) = 7.
- Initially each knows only its links: D_x = (0, 2, 7), D_y = (2, 0, 1), D_z = (7, 1, 0) as costs to (x, y, z).
- After one exchange: D_x(z) = min(7, c(x,y) + D_y(z)) = min(7, 2 + 1) = **3** (via y). D_z(x) = min(7, 1 + 2) = **3** (via y). D_y is unchanged.
- One more exchange changes nothing ⇒ **converged**: x and z reach each other via y at cost 3.

**N4. Count-to-infinity.** c(x, y) = 4, c(y, z) = 1, c(x, z) = 50. Converged: D_y(x) = 4 (direct), D_z(x) = 5 (via y). Now **c(x, y) jumps to 60**.
- y recomputes: min(60 direct, c(y,z) + D_z(x) = 1 + 5 = 6) ⇒ **6 via z**. That's wrong, since z's 5 was *through y*. Now a routing loop y ↔ z exists for x.
- z: min(50, 1 + 6) = 7 (via y). y: min(60, 1 + 7) = 8. z: 9 … each exchange adds ~1.
- The loop ends only when z's cost via y exceeds 50 and z switches to its direct link: **~44 exchanges** of packets looping between y and z meanwhile.
- **With poison reverse:** z routes to x via y, so it tells y "D_z(x) = ∞". When the link breaks, y sees only 60 (direct) ⇒ uses 60; z then picks min(50, 61) = 50 direct; y updates to min(60, 51) = 51 via z. **Converges in ~2 exchanges.**

**N5. ECMP collisions.** 4 equal paths, 2 elephant flows hashed uniformly at random.
- P(both on the same path) = 4 × (1/4)² = **1/4**. Then one path carries 2 flows while 2 paths are idle, so each elephant gets half its potential throughput.
- With 8 elephants on 8 paths: P(no collision) = 8!/8⁸ ≈ **0.24 %**. Collisions are almost certain, which is why AI-cluster fabrics use adaptive routing or packet spraying [NVIDIA].

**N6. BGP route choice.** Router R learns two routes to 203.0.113.0/24:
- Route 1: AS-PATH [65001 65010], local-pref 100.
- Route 2: AS-PATH [65002 65003 65010], local-pref 200 (a customer link, which the operator prefers).
⇒ **Route 2**: local preference is checked **before** AS-path length. With equal local-pref, Route 1 would win (shorter AS path: 2 vs 3).

**Try it:** (a) Table: 172.16.0.0/12 → A, 172.16.4.0/22 → B, 172.16.5.0/24 → C, default → D. Where do 172.16.5.9, 172.16.6.1, 172.20.0.1 and 172.40.0.1 go? (b) Dijkstra from A: A–B 4, A–C 1, C–B 2, B–D 1, C–D 5. What is A's next hop for D?
<details><summary>Answer</summary>(a) .5.9 ⇒ C (/24); .6.1 ⇒ B (/22 covers 4–7); 172.20.0.1 ⇒ A (/12 covers 172.16–172.31); 172.40.0.1 ⇒ D. (b) Costs: C 1, B 3 (via C), D 4 (A → C → B → D) ⇒ next hop <b>C</b>.</details>

---

## 4. Interview Questions

### Basic
- **Q: Routing vs forwarding?** Forwarding: per packet, look up the table and send out a port (ns, hardware, data plane). Routing: compute the tables with protocols (control plane, seconds).
- **Q: What is longest-prefix match and why is it needed?** Choose the most specific matching prefix. CIDR aggregation creates overlapping prefixes, and the specific one carries the exception.
- **Q: What is an autonomous system?** A network under one administration with its own AS number. OSPF/IS-IS run inside; BGP runs between.

### Intermediate
- **Q: Distance vector vs link state?** §2.5 table. Neighbours' vectors + Bellman-Ford vs flooded full map + Dijkstra; slow, count-to-infinity vs fast convergence.
- **Q: What is count-to-infinity and how is it mitigated?** After a link worsens, routers keep routing through each other with slowly increasing costs (N4). Mitigations: split horizon, poison reverse, a max metric (RIP 16); BGP avoids it with AS paths.
- **Q: OSPF vs BGP?** OSPF: link state, inside one AS, optimises cost. BGP: path vector, between ASes, optimises policy/business, with AS-path loop prevention, over TCP 179.
- **Q: What is ECMP and why hash per flow?** Spread traffic across equal-cost paths; hash the 5-tuple so one flow stays on one path and avoids TCP reordering.

### Advanced / killer follow-ups
- **Q [DESHAW] [AMZN]: Implement longest-prefix match.** A binary trie over address bits; store the next hop at prefix-end nodes; on lookup, walk the IP's bits and remember the last next hop seen. O(32) per lookup for IPv4:
  ```cpp
  struct Node { Node* child[2] = {nullptr, nullptr}; int hop = -1; };
  void insert(Node* root, uint32_t prefix, int len, int hop) {
      Node* n = root;
      for (int i = 0; i < len; i++) {
          int b = (prefix >> (31 - i)) & 1;
          if (!n->child[b]) n->child[b] = new Node();
          n = n->child[b];
      }
      n->hop = hop;                         // len 0 ⇒ default route stored at the root
  }
  int lookup(Node* root, uint32_t ip) {
      int best = root->hop; Node* n = root;
      for (int i = 0; i < 32 && n; i++) {
          n = n->child[(ip >> (31 - i)) & 1];
          if (n && n->hop != -1) best = n->hop;   // longer match found
      }
      return best;
  }
  ```
  *Follow-ups:* "Make it faster?" Path-compressed (Patricia) or multibit tries (stride 8 ⇒ ≤ 4 memory accesses), or TCAM in hardware. "Memory?" Compression; share subtrees.
- **Q: Why is BGP not shortest-path?** Business policy (customer > peer > provider; no free transit) comes first via local preference. AS-path length is only a tie-breaker, and an AS hop says nothing about the real distance.
- **Q: How can someone hijack traffic with BGP, and how do you defend?** Announce a victim's prefix, or a more specific one (LPM wins). Defences: RPKI route-origin validation, prefix filters, monitoring.
- **Q [AMZN]: How does a user's packet reach the nearest AWS edge or CloudFront location?** Anycast: many edge sites announce the same prefix; BGP delivers each user to a topologically close one. Combined with DNS-based routing (note 08).
- **Q [NVIDIA]: Why can ECMP perform badly for distributed training traffic?** A few huge, long-lived flows (all-reduce) ⇒ hash collisions leave links overloaded or idle (N5). Fixes: adaptive routing, packet spraying + reorder-tolerant transport, careful flow placement.
- **Q: Why does routing-protocol traffic run over TCP (BGP) or with sequence numbers (OSPF LSAs)?** The control messages must arrive reliably and in order. Stale or duplicate information would corrupt the tables.

---

## 5. Common Traps
- Picking the first matching route instead of the **longest** one.
- Forgetting that the default route /0 matches everything (it's the last resort, not an error).
- Confusing RIP (distance vector) with OSPF (link state).
- Saying BGP picks the shortest path: local preference (policy) comes first.
- Dijkstra: finalising a node before it has the smallest tentative distance, or not relaxing neighbours after adding a node.
- Thinking poison reverse solves every loop. It fixes only 2-node loops.
- Per-packet load balancing for TCP traffic (reordering ⇒ spurious retransmits).

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Forwarding vs routing | per-packet lookup (data plane, TCAM) vs table computation (control plane) |
| LPM | most specific matching prefix; /0 default; trie in SW, TCAM in HW |
| AS | one administration; intra (OSPF, IS-IS, RIP) vs inter (BGP) |
| Bellman-Ford | D_x(y) = min_v {c(x,v) + D_v(y)}; neighbours' vectors |
| Count-to-infinity | bad news slow; split horizon, poison reverse, RIP 16 = ∞ |
| Link state | flood LSAs, full map, Dijkstra, first hop → table; OSPF areas |
| DV vs LS | rumour/slow/cheap vs map/fast/robust |
| BGP | path vector over TCP 179; AS-path loop check; policy (local-pref > AS-path length) |
| BGP risks | hijacks/leaks (more specific wins); RPKI |
| Anycast | same prefix from many sites ⇒ nearest |
| ECMP | hash 5-tuple per flow; per-packet ⇒ reordering; elephant collisions |
| Router | RIB (control) → FIB (hardware); buffers; SDN = central controller |
