# 09. Instruction-Level Parallelism — Superscalar, Out-of-Order, VLIW

> **Scope:** written for **software / system-software roles**. Hardware-design depth (Tomasulo worked tables, ROB entry internals, renaming implementations, issue-queue internals, physical-register freeing) was removed; the full version is saved in [archive/09-ILP-full-hardware-depth.md](archive/09-ILP-full-hardware-depth.md) if you ever need it.
> **Asked at:** [NVIDIA] [APPLE] [QCOM] concept questions: what OoO is, renaming, ROB, precise exceptions, speculation · [TI] [QCOM] VLIW DSPs, loop unrolling · [HFT] [DESHAW] dependency chains, multiple accumulators, store-to-load forwarding, why OoO hides only some cache misses, Spectre/Meltdown awareness.
> **Prerequisites:** [04 Pipelining](04-Pipelining-and-Hazards.md) (especially §2.8 RAW/WAR/WAW and §2.17 in-order vs OoO), [05 Branch Prediction](05-Branch-Prediction.md), [06 Cache](06-Cache-Memory.md).

---

## 1. Intuition

### 1.1 The problem with in-order pipelines
The 5-stage pipeline ([04](04-Pipelining-and-Hazards.md)) has two limits:
1. **At most 1 instruction per cycle** (IPC ≤ 1).
2. **One stalled instruction blocks everything behind it**, even instructions that don't depend on it.
```
lw  x1, 0(x2)     # misses in cache: 100+ cycles
add x3, x1, x4    # needs x1: must wait (unavoidable)
sub x5, x6, x7    # independent! but in an in-order pipeline it waits behind the add
mul x8, x5, x5    # independent of the load, also waits
```
Programs contain lots of independent work like `sub` and `mul`. **Instruction-Level Parallelism (ILP)** is about finding and running it.

### 1.2 Two ideas
1. **Superscalar:** fetch, decode and execute **several instructions per cycle** (width W = 2…10).
2. **Out-of-order (OoO) execution:** look ahead through a large **window** of upcoming instructions and execute each one **as soon as its operands are ready**, not in program order. Then **commit (retire) in program order**, so the program can't tell anything unusual happened.

### 1.2.1 Superscalar explained
**Scalar pipeline (what note 04 describes):** every stage handles **one** instruction per cycle, so at most **one instruction finishes per cycle** (ideal CPI = 1, IPC = 1).
```
Cycle:  1   2   3   4   5   6
I1      IF  ID  EX  MEM WB
I2          IF  ID  EX  MEM WB
I3              IF  ID  EX  MEM WB      → one instruction enters (and later finishes) per cycle
```
**Superscalar pipeline (2-wide):** every stage handles **two** instructions per cycle, side by side, so up to **two instructions finish per cycle** (ideal CPI = 0.5, IPC = 2).
```
Cycle:  1   2   3   4   5   6
I1      IF  ID  EX  MEM WB
I2      IF  ID  EX  MEM WB              ← I1 and I2 travel together, in two "lanes"
I3          IF  ID  EX  MEM WB
I4          IF  ID  EX  MEM WB          ← I3 and I4 together
I5              IF  ID  EX  MEM WB
I6              IF  ID  EX  MEM WB      → two instructions enter (and later finish) per cycle
```
**"Width W"** = how many instructions per cycle each stage can handle. A "4-wide" core can fetch, decode and commit up to 4 instructions per cycle (ideal CPI = 0.25). Modern big cores are 4–10 wide.

**Analogy:** pipelining is an **assembly line**: each station works on a different car. Superscalar is **several assembly lines side by side**: twice the lines, twice the cars per hour (if there's enough work).

**What hardware must be duplicated for 2-wide:**
| Stage | Scalar | 2-wide superscalar |
|---|---|---|
| Fetch | 1 instruction (4 B) per cycle | 2 instructions (8 B) per cycle |
| Decode | 1 decoder | 2 decoders |
| Register file | 2 read + 1 write ports | **4 read + 2 write ports** |
| Execute | 1 ALU | 2 ALUs (or more units) |
| Forwarding | between stages | between stages **and between the two lanes** |
| Extra check | — | are the two instructions in the same cycle **independent**? |

**The catch: instructions can only go together if they're independent.**
```
add x1, x2, x3
sub x4, x1, x5     ← needs x1 from the add → can't execute in the same cycle → one lane sits empty
```
Example: a **2-wide in-order** core (all ops take 1 cycle; a dependent instruction can go the cycle after its producer; instructions issue in program order).
```
I1: add x1, x2, x3
I2: sub x4, x5, x6      independent of I1          → issues with I1
I3: and x7, x1, x4      needs I1 and I2            → next cycle
I4: or  x8, x7, x9      needs I3                   → can't pair with I3, next cycle
I5: xor x10, x11, x12   independent                → pairs with I4
I6: add x13, x10, x8    needs I4 and I5            → next cycle
```
| Cycle | Lane 1 | Lane 2 |
|---|---|---|
| 1 | I1 | I2 |
| 2 | I3 | *(empty: I4 depends on I3)* |
| 3 | I4 | I5 |
| 4 | I6 | *(empty: nothing left)* |
6 instructions in 4 cycles ⇒ **IPC = 1.5**: better than scalar (6 cycles, IPC 1), but below the ideal 2. Real code has dependences, branches and cache misses, so real IPC is well below the width (e.g., ~1.5–4 on 4–8-wide cores).

**Why superscalar and out-of-order go together:** an **in-order** superscalar core can only pair *neighbouring* instructions (above, lanes stayed empty). An **out-of-order** core looks far ahead in its window and finds independent instructions to fill every lane. In-order superscalar examples: original Pentium, ARM Cortex-A53/A55 (2-wide). OoO superscalar: Intel, AMD, Apple big cores.

**OoO does not require width > 1.** Width and order are independent choices ([04 §2.17](04-Pipelining-and-Hazards.md)). A width-1 OoO core issues and commits 1 instruction per cycle (IPC ≤ 1), but it can pick a younger independent instruction while an older one is stuck, and several long operations can be in progress at once. Example (miss = 100 cycles): `lw x1; add (needs x1); lw x4 (independent); add (needs x4)`. In-order width 1: the second load waits behind the stalled add, so the misses happen back to back (≈ 200 cycles). OoO width 1: the second load issues during the first miss (≈ 105 cycles). OoO hides latency; it doesn't raise peak IPC.
| | Width 1 (scalar) | Width > 1 (superscalar) |
|---|---|---|
| **In-order** | classic 5-stage, Cortex-M4 | Cortex-A53/A55, original Pentium |
| **Out-of-order** | CDC 6600, IBM 360/91 (historic) | modern Intel, AMD, Apple big cores |
Width-1 OoO cores are rare today only because OoO hardware is expensive: once it finds independent instructions, designers add width to run more of them per cycle.

**Superscalar vs similar-sounding ideas:**
| | What runs in parallel | Who decides |
|---|---|---|
| Pipelining | different **stages** of different instructions | hardware |
| **Superscalar** | several **whole instructions** in the same stage, same cycle | hardware, at run time |
| VLIW | several operations packed into **one** long instruction | compiler (§2.2) |
| SIMD | **one** instruction working on many data elements ([11](11-SIMD-Multithreading-Multicore.md)) | programmer / compiler |
| Multicore | separate threads/programs on separate cores | OS / programmer |

### 1.3 Analogy: a restaurant kitchen
- Orders (instructions) arrive at the counter **in order** and are written on a ticket rail (**ROB**).
- Cooks (**execution units**) prepare **whichever dish has all its ingredients ready**, not necessarily the oldest order. A dish waiting for bread from the bakery (**cache miss**) doesn't stop the cooks from making salads for later orders.
- The waiter (**commit**) serves dishes **strictly in order of the tickets**. If ticket 1 isn't ready, tickets 2–5 wait on the pass even though they're cooked.
- If an order is cancelled (**branch mispredict**), every dish made for later tickets is thrown away (**squash**).

### 1.4 Vocabulary
| Term | Meaning |
|---|---|
| **ILP** | Number of instructions that could run in parallel, given their dependences |
| **Width (W)** | Instructions fetched/decoded/committed per cycle |
| **Window** | The set of in-flight instructions the core can choose from (≈ ROB size) |
| **Issue** | Send an instruction whose operands are ready to an execution unit (**out of order**) |
| **Commit / retire** | Make the result architecturally permanent, in program order |
| **ROB** (reorder buffer) | In-order list of all in-flight instructions; enforces in-order commit |
| **RAT** (register alias table) | Maps each architectural register to its current physical register |
| **Physical registers** | A large pool of hidden registers (e.g., ~280) behind the 16/32 architectural ones |
| **Free list** | Physical registers not currently in use |
| **Speculation** | Executing instructions before knowing they should run (e.g., after a predicted branch) |
| **Squash / flush** | Discard wrong-path instructions and undo their effects |

---

## 2. Deep Dive

### 2.1 What limits ILP
| Limit | Example | Removed / reduced by |
|---|---|---|
| **True (RAW) data dependences** | `add x3, x1, …` needs `x1` from the load | can't be removed: this is the **dataflow limit** (critical path). Only hidden by doing other work meanwhile |
| **Name dependences (WAR, WAW)** | reusing `x1` for a new value | **register renaming** (2.6) |
| **Control dependences** | instructions after a branch | **branch prediction + speculation** (2.10) |
| **Memory dependences** | does this load read what that store wrote? | **memory disambiguation** (2.9) |
| **Resources** | width, number of ALUs, ROB size, power | more hardware (diminishing returns) |

### 2.2 Static (compiler) ILP
The compiler can expose and schedule independent work before the program runs.

**Loop unrolling + scheduling** (H&P classic). `for (i) x[i] = x[i] + s;` (double). Latencies: load→FP add 1 stall, FP add→store 2 stalls, integer ALU→branch 1 stall.
| Version | Cycles | Cycles / element |
|---|---|---|
| Naive | 9 (L.D, stall, ADD.D, stall, stall, S.D, ADDI, stall, BNE) | 9 |
| Scheduled | 7 | 7 |
| Unrolled 4× (renamed registers), unscheduled | 27 | 6.75 |
| **Unrolled 4× + scheduled** | **14** | **3.5** |
Unrolling removes loop overhead (fewer increments and branches) and gives the scheduler 4 independent iterations to interleave. Costs: larger code, more registers needed (**register pressure**).

**VLIW (Very Long Instruction Word)** [TI][QCOM]: the compiler bundles independent operations into one wide instruction, and the hardware issues the whole bundle with **no dependency checking**.
| | Superscalar OoO | VLIW |
|---|---|---|
| Who finds parallelism | hardware, at run time | compiler, at compile time |
| Hardware complexity / power | high | low |
| Handles unpredictable latency (cache misses) | well | poorly (stalls whole bundle) |
| Binary compatibility across widths | yes | poor |
| Where it wins | general-purpose CPUs | **DSPs** (TI C6000 8-wide, Qualcomm Hexagon 4-wide), some AI accelerators |
VLIW compilers rely on **software pipelining**: each loop iteration overlaps parts of several original iterations (store for i, add for i+1, load for i+2), so the body has no stalls. Itanium is the famous general-purpose VLIW failure: compilers couldn't predict cache misses.

### 2.3 Why do it in hardware?
The compiler can't know at compile time:
- which loads will **miss in the cache** (latency varies from 4 to 300+ cycles),
- which way **branches** go in this run,
- whether two pointers **point to the same address**.
Hardware sees these at run time, and the same binary runs well on cores of different widths. The price is power and area, which is why efficiency cores and GPUs stay in-order.

### 2.4 The big picture: an OoO pipeline
```
 IN ORDER (front end)                      OUT OF ORDER (back end)                     IN ORDER
 ┌───────┐  ┌────────┐  ┌────────┐  ┌──────────┐     ┌──────────────┐   ┌──────────┐   ┌────────┐
 │ Fetch │─►│ Decode │─►│ Rename │─►│ Dispatch │──┬─►│ Issue queue  │──►│ Execute  │──►│ Commit │
 │(+pred)│  │        │  │ (RAT)  │  │          │  │  │ (wait for    │   │ ALU / FP │   │(retire)│
 └───────┘  └────────┘  └────────┘  └──────────┘  │  │  operands)   │   │ AGU / LSU│   └────────┘
                                         │        │  └──────────────┘   └────┬─────┘       ▲
                                         │        └─► Load/Store queue ──────┘             │
                                         └──────────► ROB (one entry per instruction, in program order) ─┘
```
| Structure | Role |
|---|---|
| **RAT** | maps architectural reg → current physical reg (renaming) |
| **Physical registers** + free list | hold all values, speculative and committed |
| **ROB** | in-order list of in-flight instructions; commit in order ⇒ precise exceptions, recovery from mispredicts |
| **Issue queue** | instructions wait here until their operands are ready |
| **Load/store queue (LSQ)** | tracks memory operations and their ordering; store-to-load forwarding |

### 2.5 Walkthrough: one OoO execution, cycle by cycle
**Code:**
```
I1: lw   x1, 0(x2)      # cache miss: 10 cycles
I2: add  x3, x1, x4     # depends on I1
I3: sub  x5, x6, x7     # independent
I4: mul  x8, x5, x5     # depends on I3 (3 cycles)
I5: addi x9, x9, 1      # independent
```
**Simplified model:** dispatch 1 instruction per cycle (in order); an instruction issues the cycle after dispatch at the earliest, and only when its operands are ready; a result can be used the cycle after it's produced; commit is in order, up to 4 per cycle, at the earliest the cycle after completion. Latencies: load miss 10, mul 3, others 1.

Legend: **D** = dispatched into ROB + issue queue · **w** = waiting in issue queue for an operand · **E** = executing · **r** = finished, waiting in ROB to commit · **C** = committed
```
Cycle:           1  2  3  4  5  6  7  8  9  10 11 12 13
I1 lw   x1,0(x2) D  E  E  E  E  E  E  E  E  E  E  C
I2 add  x3,x1,x4    D  w  w  w  w  w  w  w  w  w  E  C
I3 sub  x5,x6,x7       D  E  r  r  r  r  r  r  r  r  C
I4 mul  x8,x5,x5          D  E  E  E  r  r  r  r  r  C
I5 addi x9,x9,1              D  E  r  r  r  r  r  r  C
```
What happened:
1. **Cycles 2–11:** the load waits on memory. I2 needs `x1`, so it waits in the issue queue (`w`).
2. **Cycles 4–7:** I3, I4, I5 **execute out of order, during the miss**. They're younger than I2 but don't depend on it. This is the whole point of OoO.
3. **Cycles 5–12:** I3, I4, I5 are done but **can't commit**, because I1 (the oldest) isn't done. They wait in the ROB (`r`).
4. **Cycle 12:** I1 commits. **Cycle 13:** I2 commits, and I3, I4, I5 commit in the same cycle, in order (commit width 4).

**Same code on an in-order pipeline** (same latencies, issue strictly in order, 1 per cycle):
| | I1 | I2 | I3 | I4 | I5 |
|---|---|---|---|---|---|
| Executes (in-order) | 2–11 | 12 | 13 | 14–16 | 15 |
| Executes (out-of-order) | 2–11 | 12 | **4** | **5–7** | **6** |
In-order: I3 can't start until I2 starts, so everything finishes at about **cycle 17**. OoO finishes at **cycle 13**: the work of I3–I5 was hidden under the miss.

### 2.6 Register renaming
**Why rename?** There are only 32 architectural registers (16 on x86-64), so compilers reuse register names constantly. That creates WAR and WAW dependences that are **not real data flow** ([04 §2.8](04-Pipelining-and-Hazards.md)). Renaming gives every new value its own **physical register**, so only true (RAW) dependences remain.

**Code** (note `x1` is reused by I3):
```
I1: lw   x1, 0(x2)
I2: add  x3, x1, x4     RAW on x1 (I1 → I2): real
I3: sub  x1, x5, x6     WAW on x1 (with I1), WAR on x1 (with I2): name-only
I4: mul  x7, x1, x3     RAW on x1 (I3 → I4) and x3 (I2 → I4)
```
**Rename rules, for each instruction in order:**
1. **Sources:** look up each source register in the RAT → use that physical register.
2. **Destination:** take a new physical register from the **free list**.
3. **Update the RAT:** destination → new physical register.

**Initial state:** RAT `x1→P1, x2→P2, x3→P3, x4→P4, x5→P5, x6→P6, x7→P7`; free list = P8, P9, P10, P11, …
| Step | Original | Renamed | RAT after (x1, x3, x7) |
|---|---|---|---|
| start | | | P1, P3, P7 |
| I1 | `lw x1, 0(x2)` | `lw P8, 0(P2)` | **P8**, P3, P7 |
| I2 | `add x3, x1, x4` | `add P9, P8, P4` | P8, **P9**, P7 |
| I3 | `sub x1, x5, x6` | `sub P10, P5, P6` | **P10**, P9, P7 |
| I4 | `mul x7, x1, x3` | `mul P11, P10, P9` | P10, P9, **P11** |

Look at the renamed code:
```
lw  P8,  0(P2)
add P9,  P8, P4      ← reads P8 (I1's result): RAW kept ✅
sub P10, P5, P6      ← writes P10, not P8: no conflict with I1 or I2 → can run in parallel with I1!
mul P11, P10, P9     ← reads P10 (I3's x1), not P8: correct value ✅
```
The WAR and WAW on `x1` are gone, and I3 no longer waits for anything. On a branch mispredict, the RAT is restored to a copy (checkpoint) saved at the branch, so wrong-path renamings disappear.

### 2.7 The reorder buffer (ROB)
**A circular buffer of in-flight instructions, in program order:**
```
   head (oldest: next to commit)                     tail (next free entry: newest)
     ▼                                                 ▼
 ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
 │  I1  │  I2  │  I3  │  I4  │  I5  │  I6  │ free │ free │ free │
 │ wait │ done │ done │ done │ exec │ wait │      │      │      │
 └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
 I1 is not done → NOTHING commits this cycle, even though I2–I4 are done.
```
- New instructions enter at the **tail** (in program order).
- **Commit**, each cycle: if the **head** instruction is done with no exception, retire it (make its result permanent; a store is released to the cache). Repeat for up to W instructions.
- **ROB full** ⇒ no new instructions can enter ⇒ the core stalls. A long cache miss at the head is the usual cause (N4).

**Precise exceptions:** an exception is recorded in the instruction's ROB entry and taken only when it reaches the **head**. Then everything younger is flushed and the register mapping is restored to the committed state. Result: everything before the faulting instruction is done, nothing after it is, which is what the OS needs to handle a page fault and restart the instruction. Wrong-path instructions that faulted are squashed before reaching the head, so their exceptions never happen.

**Branch mispredict:** as soon as the branch resolves wrong, flush the ROB entries **younger** than the branch, restore the RAT checkpoint, and fetch from the correct path.

### 2.8 Tomasulo and scoreboarding (concept level)
These are the two classic dynamic-scheduling algorithms from textbooks. You should be able to explain them in a few sentences:
- **Scoreboarding** (CDC 6600, 1960s): a central table tracks which instruction will write which register. Instructions execute when operands are ready, but there's **no renaming**, so WAR and WAW hazards cause **stalls**.
- **Tomasulo's algorithm** (IBM 360/91): instructions wait in **reservation stations**. A source operand is either copied in as a **value** (if ready) or recorded as a **tag** naming the station that will produce it. Results are **broadcast on a common data bus (CDB)**, and every waiting station with that tag grabs the value. Tags act as renaming, so **WAR and WAW disappear**, and broadcasting gives forwarding for free.
- Original Tomasulo had **no ROB**, so exceptions were imprecise. Adding a ROB with in-order commit gives the modern OoO core.

| | Scoreboard | Tomasulo | Modern OoO |
|---|---|---|---|
| WAR / WAW | stall | eliminated (tags) | eliminated (renaming) |
| Forwarding | no | yes (CDB) | yes |
| Precise exceptions | no | no | yes (ROB) |

### 2.9 Loads and stores out of order
Registers are renamed, but **memory addresses can't be renamed**, and an address is known only after it's computed.
```
sw  x5, 0(x6)     # store: x6 comes from a slow instruction → address unknown for a while
lw  x7, 0(x8)     # load: x8 is ready → can it execute before the store?
```
- If the addresses differ, the load can go early. ✅ If they're the same, the load must get `x5` from the store. ❌
- **Modern cores guess:** run the load early. When the store's address is known, check whether they overlapped; if so, flush the load and everything after it and re-execute. A predictor learns which loads tend to conflict and makes only those wait.
- **Store-to-load forwarding** [HFT]: if a load reads an address that an older, not-yet-committed store wrote, it gets the data directly from the **store queue** (~4–5 cycles). Partial overlaps (e.g., an 8-byte load from two separate 4-byte stores) may fail to forward ⇒ stall ~10+ cycles. Writing a field and immediately reading it back with a different size is a classic hidden slowdown.
- **Stores write the cache only after commit.** A store can't be undone once in memory (another core could see it), so it waits in the store queue/buffer until it's no longer speculative. (This store buffer is also why x86 allows store→load reordering: see [10](10-Cache-Coherence-and-Memory-Consistency.md).)

### 2.10 Speculation: what's safe, and Spectre vs Meltdown
**Speculative instruction:** one that might not be on the correct path (after an unresolved predicted branch) or that might be cancelled (after an older instruction that might fault). The core runs them anyway, but keeps their effects **invisible** until commit.
| Can be done speculatively | Only at commit (or once non-speculative) |
|---|---|
| Compute into physical registers | Update the architectural register state |
| Read the caches (loads) | **Write** the cache/memory (stores) |
| Update branch history (repaired on mispredict) | Raise exceptions |
| | Access **MMIO / device memory** (reading a UART FIFO pops data; it can't be undone) [TI] |
Device memory is marked **non-speculative** in the page tables / MPU attributes, so the core never touches it on a wrong path.

**The catch:** squashing restores registers, but **not the cache**. A wrong-path load leaves a footprint (which line was brought in), and an attacker can measure that by timing. That's the root of transient-execution attacks:
| | **Spectre v1** | **Meltdown** |
|---|---|---|
| Trigger | branch **misprediction** (e.g., a bounds check predicted "in bounds") | a load that **faults** (user mode reading kernel memory), whose data reaches dependent instructions **before** the fault is taken at commit |
| What leaks | data the victim process can legally address | **kernel memory** from user space |
| Affected | almost all speculative CPUs (Intel, AMD, ARM, Apple…) | mainly Intel (also some ARM and IBM POWER cores); AMD not affected |
| Mitigation | `lfence`/speculation barriers, index masking, compiler fixes | **KPTI** (kernel page tables unmapped from user space); fixed in later hardware |
See also [05 §2.9](05-Branch-Prediction.md).

### 2.11 Why OoO helps with cache misses, and its limit
While a load misses (say 300 cycles), the core keeps executing independent younger instructions until the **ROB fills**. A 512-entry ROB at 4 instructions/cycle fills in ~128 cycles. So OoO can **overlap independent misses** (memory-level parallelism, MLP), but it can't hide a single 300-cycle miss completely (N4). **Dependent misses** (pointer chasing: each address comes from the previous load) get no help at all (N3). Typical modern ROB sizes are roughly 300–600 entries.

### 2.12 Software consequences [HFT][DESHAW][NVIDIA]
**Dependency chains limit throughput, not instruction count:**
```c
double s = 0;                 // FP add latency 4 cycles, 2 FP add units
for (i) s += a[i];            // chain: 1 add per 4 cycles → 0.25 adds/cycle

double s0=0,s1=0,s2=0,s3=0,s4=0,s5=0,s6=0,s7=0;   // 8 independent chains
for (i += 8) { s0 += a[i]; s1 += a[i+1]; ... }    // up to 2 adds/cycle (latency × throughput = 4 × 2 = 8 chains)
```
(Compilers won't do this for floats without `-ffast-math` because it changes rounding; see [01](01-Number-Representation-and-Arithmetic.md).)
- Avoid long serial chains (pointer chasing, hash chains); interleave independent work.
- `std::unordered_map` lookups = dependent loads; open addressing with prefetch can overlap them.
- Independent cache misses close together in the code overlap (MLP); misses separated by long dependent chains don't.
- Avoid store-to-load forwarding failures (writing a value and reading it back with a different size/offset).

---

## 3. Worked Numericals

**N1. Chains needed to saturate units.** FMA latency 4 cycles, 2 FMA units/cycle ⇒ independent accumulators needed = latency × throughput = **8** (with AVX-512, each is a vector ⇒ 8 × 16 floats in flight).

**N2. Unrolling speedup.** From the table in 2.2: 9 → 3.5 cycles/element = **2.57×** speedup with zero hardware change.

**N3. Independent vs dependent misses (MLP).** Miss latency 200 cycles, everything else 1 cycle.
- **Independent:** `lw x1, 0(x2); lw x3, 0(x4); add x5, x1, x3`. OoO issues both loads back to back (cycles 2 and 3); data returns at ~202 and ~203; add at ~204 ⇒ **≈ 204 cycles**. A **blocking** in-order core (stalls on every miss) ⇒ ≈ 400+.
- **Dependent (pointer chase):** `lw x1, 0(x2); lw x3, 0(x1)`. The second address comes from the first load ⇒ **≈ 400 cycles on any core**.
- Nuance: an in-order core with a **non-blocking cache** that stalls only when a value is *used* would also overlap the two independent loads above. OoO's real advantage is finding independent misses **much further ahead** (up to the ROB size), even past stalled instructions.

**N4. How much of a miss can the ROB hide?** ROB = 224 entries, 4 instructions enter per cycle, a load at the ROB head misses for 300 cycles, and everything after it is independent.
- The other 223 entries fill in 223/4 ≈ **56 cycles**; then the ROB is full and the core stops accepting instructions.
- The core stalls for the remaining 300 − 56 ≈ **244 cycles**. Only ~**19%** of the miss is hidden.
- If another independent miss appears within those 223 instructions, it overlaps with the first one, which is the real benefit.

**N5. In-order vs OoO timing.** Same model as 2.5 (dispatch 1/cycle, commit up to 4/cycle). Load miss 6 cycles, others 1.
```
I1: lw   x1, 0(x2)
I2: add  x3, x1, x1
I3: addi x4, x4, 1
I4: addi x5, x5, 1
I5: add  x6, x4, x5
```
| | I1 | I2 | I3 | I4 | I5 | Done (last commit) |
|---|---|---|---|---|---|---|
| OoO execute | 2–7 | 8 | 4 | 5 | 6 | I1 commits 8; I2–I5 commit 9 ⇒ **9** |
| In-order execute | 2–7 | 8 | 9 | 10 | 11 | **12** |
Speedup = 12 / 9 = **1.33×**, because I3–I5 ran during the miss.

---

## 4. Interview Questions

### Basic
- **Q: Superscalar vs pipelined?** Pipelined overlaps stages (IPC ≤ 1); superscalar issues multiple instructions per cycle (IPC > 1).
- **Q: What is out-of-order execution?** Execute each instruction when its operands are ready instead of in program order; commit in order so the result is identical.
- **Q: What is register renaming and what hazards does it remove?** Map each new value to a fresh physical register; removes WAR and WAW (not RAW).
- **Q: Is an OoO core out of order everywhere?** No: in-order front end (fetch → dispatch), out-of-order execution, in-order commit.

### Intermediate
- **Q: Why do we need a ROB?** In-order commit ⇒ precise exceptions, squash on mispredict, stores written only after commit.
- **Q: How does an OoO core give precise exceptions?** Exceptions are recorded in the ROB and taken only at the head; younger instructions are flushed.
- **Q: What happens on a branch mispredict?** Flush instructions younger than the branch, restore the register mapping, fetch from the correct path.
- **Q: Scoreboarding vs Tomasulo?** Scoreboard stalls on WAR/WAW and has no forwarding; Tomasulo removes them using tags (renaming) and broadcasts results on the CDB (2.8).
- **Q [TI][QCOM]: VLIW vs superscalar? Why do DSPs use VLIW?** Regular, predictable loops (filters, FFT) let the compiler schedule perfectly ⇒ high throughput at low power.
- **Q: What is loop unrolling and its downsides?** (2.2.)

### Advanced / killer follow-ups
- **Q [APPLE]: Can a load execute before an earlier store whose address is unknown?** Yes, speculatively; if they turn out to overlap, the load and younger instructions are re-executed (2.9).
- **Q [HFT]: What is store-to-load forwarding and when does it fail?** Load gets data from the store queue; fails on partial overlap or size mismatch ⇒ ~10+ cycle stall.
- **Q: Why doesn't a bigger ROB keep helping?** Mispredicts throw away the window, dependent misses can't overlap, and large structures cost power and clock speed.
- **Q: Why can't MMIO reads be executed speculatively?** Reading a device register can have side effects (popping a FIFO, clearing a flag); device memory is marked non-speculative.
- **Q: Spectre vs Meltdown?** (2.10.) Both exploit the cache footprint of squashed instructions; Spectre via mispredicted branches, Meltdown via data forwarded from a faulting load. Meltdown's OS fix is KPTI.
- **Q [NVIDIA]: Why are GPU cores in-order while CPUs are OoO?** GPUs target throughput: thousands of threads hide latency, which is far cheaper per FLOP than OoO hardware built for single-thread latency.
- **Q [HFT]: Your summation loop runs at 1/8 of peak FLOPs. Why, and fix?** A single accumulator forms a dependency chain; use multiple accumulators / SIMD; mind FP reassociation.
- **Q [HFT]: Will OoO speed up traversing a linked list where each node misses?** No: each address depends on the previous load, so misses are serialised (N3). Use contiguous arrays or prefetching.

---

## 5. Common Traps
- Saying renaming removes RAW. It doesn't; only WAR and WAW.
- Saying OoO commits out of order. It **executes** out of order and **commits in order**.
- Thinking OoO hides all memory latency (N4) or helps pointer chasing (N3).
- Forgetting that stores update memory only at commit.
- Thinking squashing undoes everything. It undoes registers, but **not cache state** (Spectre/Meltdown).

---

## 6. Self-Check Quiz
1. In the 2.5 walkthrough, why can't I3 commit at cycle 5, even though it finished at cycle 4?
2. Rename this code. Initial RAT: `x1→P1, x2→P2, x3→P3`, free list P4, P5, P6.
   `I1: add x1, x2, x3` · `I2: sub x2, x1, x1` · `I3: add x1, x2, x1`
3. A sum loop `s += a[i]` with FP add latency 4 runs at 1 add per 4 cycles on a wide OoO core. Why doesn't OoO help, and how do you fix it?
4. A 256-entry ROB, 4 instructions enter per cycle, a 400-cycle miss at the head. Roughly how many cycles of the miss are hidden?
5. Why must a store wait until commit before writing the cache, while a load can read the cache speculatively?

<details><summary>Answers</summary>

1. Commit is in order. I1 (older) is still waiting for memory at the ROB head, so nothing behind it can commit.
2. I1: `add P4, P2, P3` (x1→P4). I2: `sub P5, P4, P4` (x2→P5). I3: `add P6, P5, P4` (x1→P6). Final RAT: x1→P6, x2→P5, x3→P3.
3. Every add needs the previous add's result (a RAW chain), so there's no independent work to reorder. Fix: several independent accumulators (latency × throughput of them) or SIMD, then combine at the end.
4. The other 255 entries fill in about 255/4 ≈ **64 cycles**; the remaining ~336 cycles stall (≈ 16% hidden).
5. A store to memory can't be undone, since other cores or devices may observe it. A speculative load changes no architectural state (only cache contents, the Spectre caveat), so it's simply discarded on a squash.
</details>

---

## 7. Revision Summary
| | |
|---|---|
| ILP limits | RAW (dataflow, can't remove), WAR/WAW (renaming), control (prediction), memory (disambiguation), resources |
| Static ILP | unrolling, scheduling; VLIW + software pipelining (TI C6000, Hexagon) |
| OoO shape | in-order front (fetch → rename → dispatch) · OoO execute · in-order commit |
| Renaming | sources via RAT; new dest from free list; update RAT ⇒ WAR/WAW gone |
| ROB | circular buffer; commit from head when done; exceptions taken at head (precise); mispredict flushes younger entries |
| Tomasulo vs scoreboard | tags + CDB remove WAR/WAW vs stalls; neither precise without a ROB |
| Memory | loads may bypass stores speculatively; store-to-load forwarding; stores at commit |
| Speculation | registers rolled back, cache not ⇒ Spectre (mispredict) / Meltdown (faulting load); MMIO never speculative |
| Miss hiding | ROB fill time = ROB / width; overlaps independent misses (MLP), not dependent |
| Code | break dependency chains (latency × throughput accumulators); avoid pointer chasing and forwarding failures |
