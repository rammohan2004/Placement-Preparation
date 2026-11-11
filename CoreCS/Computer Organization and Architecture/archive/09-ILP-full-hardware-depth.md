# 09. Instruction-Level Parallelism — Superscalar, Out-of-Order, VLIW

> **Asked at:** [APPLE] [NVIDIA] [QCOM] register renaming, ROB, Tomasulo, memory disambiguation, speculation: the "senior architecture" questions · [TI] [QCOM] VLIW DSPs (TI C6000, Qualcomm Hexagon), loop unrolling, software pipelining · [HFT] [DESHAW] dependency chains, multiple accumulators, why OoO hides some cache misses, Spectre/Meltdown awareness.
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
| **Dispatch** | Place a renamed instruction into the ROB and the scheduler (in order) |
| **Issue** | Send an instruction whose operands are ready to an execution unit (**out of order**) |
| **Writeback / complete** | Result produced and broadcast to waiting instructions |
| **Commit / retire** | Make the result architecturally permanent, in program order |
| **ROB** (reorder buffer) | In-order list of all in-flight instructions; enforces in-order commit |
| **RAT** (register alias table) | Maps each architectural register to its current physical register |
| **PRF** (physical register file) | Large register file holding all values, speculative and committed |
| **Free list** | Physical registers not currently in use |
| **Issue queue / reservation station** | Where instructions wait for their operands |
| **LSQ** (load/store queue) | Tracks in-flight memory operations and their ordering |
| **Speculation** | Executing instructions before knowing they should run (e.g., after a predicted branch) |
| **Squash / flush** | Discard wrong-path instructions and undo their effects |

---

## 2. Deep Dive

### 2.1 What limits ILP
| Limit | Example | Removed / reduced by |
|---|---|---|
| **True (RAW) data dependences** | `add x3, x1, …` needs `x1` from the load | can't be removed: this is the **dataflow limit** (critical path). Only hidden by doing other work meanwhile |
| **Name dependences (WAR, WAW)** | reusing `x1` for a new value | **register renaming** (2.6) |
| **Control dependences** | instructions after a branch | **branch prediction + speculation** (2.13) |
| **Memory dependences** | does this load read what that store wrote? | **memory disambiguation** (2.12) |
| **Resources** | width, number of ALUs, ROB size, ports, power | more hardware (with diminishing returns, 2.15) |

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

**Software pipelining:** reorganise the loop so that each new iteration contains instructions from *different* original iterations:
```
Original iteration i:   LOAD(i) → ADD(i) → STORE(i)

Software-pipelined steady-state loop body:
    STORE(i)      ← finishing iteration i
    ADD(i+1)      ← middle of iteration i+1
    LOAD(i+2)     ← starting iteration i+2
(+ a prologue before the loop and an epilogue after it to fill/drain)
```
No instruction in the body depends on another instruction in the same body, so no stalls. It's the software analogue of what OoO hardware does. TI C6000 compilers do this aggressively (**modulo scheduling**).

**VLIW (Very Long Instruction Word):** the compiler bundles independent operations into one wide instruction, and the hardware issues the whole bundle with **no dependency checking**.
| | Superscalar OoO | VLIW |
|---|---|---|
| Who finds parallelism | hardware, at run time | compiler, at compile time |
| Hardware complexity / power | high | low |
| Handles unpredictable latency (cache misses) | well | poorly (stalls whole bundle) |
| Binary compatibility across widths | yes | poor |
| Where it wins | general-purpose CPUs | **DSPs** (TI C6000 8-wide, Qualcomm Hexagon 4-wide), some AI accelerators |
Itanium (EPIC) is the famous general-purpose VLIW failure: compilers couldn't predict cache misses.

### 2.3 Why do it in hardware (dynamic scheduling)?
The compiler can't know at compile time:
- which loads will **miss in the cache** (latency varies from 4 to 300+ cycles),
- which way **branches** go in this run,
- whether two pointers **point to the same address**.
Hardware sees these at run time, and the same binary runs well on cores of different widths. The price is power and area (2.15).

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
| **RAT** (register alias table) | maps architectural reg → current physical reg |
| **Physical register file (PRF)** + free list | holds all speculative + committed values (e.g., ~280+ int regs vs 16/32 architectural) |
| **ROB** | in-order list of in-flight instructions; commit in order ⇒ **precise exceptions**, recovery from mispredicts |
| **Issue queue / reservation stations** | wait for operands; select oldest-ready each cycle (wakeup/select loop, a critical timing path) |
| **Load/store queues (LSQ)** | track memory ops, ordering, **store-to-load forwarding** |
| **Retirement (architectural) RAT** | mapping of committed state; used to recover on exceptions |

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
3. **Cycles 5–12:** I3, I4, I5 are done but **can't commit**, because I1 (the oldest, at the ROB head) isn't done. They sit in the ROB (`r`).
4. **Cycle 12:** I1 commits. **Cycle 13:** I2 commits, and I3, I4, I5 commit in the same cycle, in order (commit width 4).

**Same code on an in-order pipeline** (same latencies, issue strictly in order, 1 per cycle):
| | I1 | I2 | I3 | I4 | I5 |
|---|---|---|---|---|---|
| Executes (in-order) | 2–11 | 12 | 13 | 14–16 | 15 |
| Executes (out-of-order) | 2–11 | 12 | **4** | **5–7** | **6** |
In-order: I3 can't start until I2 starts, so everything finishes at about **cycle 17**. OoO finishes at **cycle 13**: the work of I3–I5 was hidden under the miss.

### 2.6 Register renaming, step by step
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
3. **Save the old mapping** of the destination in the instruction's ROB entry (needed later to free it, or to undo on a squash).
4. **Update the RAT:** destination → new physical register.

**Initial state:** RAT `x1→P1, x2→P2, x3→P3, x4→P4, x5→P5, x6→P6, x7→P7`; free list = P8, P9, P10, P11, …

| Step | Original | Renamed | New dest | Old mapping saved in ROB | RAT after (x1, x3, x7) | Free list after |
|---|---|---|---|---|---|---|
| start | | | | | P1, P3, P7 | P8, P9, P10, P11, … |
| I1 | `lw x1, 0(x2)` | `lw P8, 0(P2)` | P8 | P1 | **P8**, P3, P7 | P9, P10, P11, … |
| I2 | `add x3, x1, x4` | `add P9, P8, P4` | P9 | P3 | P8, **P9**, P7 | P10, P11, … |
| I3 | `sub x1, x5, x6` | `sub P10, P5, P6` | P10 | P8 | **P10**, P9, P7 | P11, … |
| I4 | `mul x7, x1, x3` | `mul P11, P10, P9` | P11 | P7 | P10, P9, **P11** | P12, … |

Look at the renamed code:
```
lw  P8,  0(P2)
add P9,  P8, P4      ← reads P8 (I1's result): RAW kept ✅
sub P10, P5, P6      ← writes P10, not P8: no conflict with I1 or I2 anymore → can run in parallel with I1!
mul P11, P10, P9     ← reads P10 (I3's x1), not P8: correct value ✅
```
The WAR and WAW on `x1` are gone. I3 no longer has to wait for anything.

**When is a physical register freed?** When the instruction that **overwrote its mapping commits**, free the **old** mapping saved in its ROB entry:
| Commit | Frees | Why it's safe |
|---|---|---|
| I1 | P1 (old `x1`) | Every instruction that could read the old `x1` is older than I1 (younger ones were renamed to P8), and all older instructions have committed |
| I2 | P3 (old `x3`) | same reasoning |
| I3 | **P8** (I1's result) | Its only reader, I2, is older than I3 and has committed |
| I4 | P7 (old `x7`) | same reasoning |
Why not free P8 as soon as I2 (its reader) *executes*? (1) The hardware doesn't track how many readers a register has. (2) If an exception or mispredict occurs between I1 and I3, the machine must roll back to a state where `x1` = P8, so P8 must still exist.

**Squash (branch mispredict) example:** suppose a mispredicted branch sat between I2 and I3. I3 and I4 are on the wrong path:
1. Return their **new** registers (P10, P11) to the free list.
2. Restore the RAT to the **checkpoint taken at the branch**: `x1→P8, x3→P9, x7→P7`.
3. Redirect fetch to the correct path.

**Superscalar renaming:** a 4-wide core renames 4 instructions **in the same cycle**. If I2 reads a register written by I1 in the same group, it must get I1's *new* physical register, not the RAT's old entry. So the rename stage compares every source against every earlier destination in the group: O(W²) comparators. This is one reason very wide cores are hard (2.15).

**Minimum physical registers** = architectural registers + max in-flight instructions that write a register (see N4).

### 2.7 Two ways to implement renaming (classic interview follow-up)
| | **ROB-based** (values stored in the ROB) | **Unified physical register file (PRF)** |
|---|---|---|
| Where speculative values live | in the ROB entry | in the PRF |
| Where committed values live | separate **architectural register file (ARF)** | same PRF (retirement RAT says which are committed) |
| RAT points to | either the ARF or a ROB entry | a physical register |
| At commit | **copy** value from ROB → ARF | only update the retirement RAT and free the old register; **no data moves** |
| Pros | simple freeing (ROB entry = register) | no copying (big win for 256/512-bit vector registers); ROB needs no data ports |
| Cons | data written twice; ROB needs many wide read/write ports | more complex free-list management |
| Used in | Intel P6 family (Pentium Pro … Nehalem) | MIPS R10000, Alpha 21264, Intel since Sandy Bridge, AMD Zen, Apple, modern ARM |
Original Tomasulo (2.10) is a third variant: tags are reservation-station numbers, and values are kept in the reservation stations and the register file.

### 2.8 The reorder buffer (ROB) in detail
**What's in a ROB entry (PRF-style design):**
| Field | Purpose |
|---|---|
| Valid | entry in use |
| PC | to restart / report the instruction on an exception |
| Type | ALU / load / store / branch |
| Arch dest, new phys reg, **old phys reg** | at commit: free the old one; on squash: free the new one |
| Done (ready) bit | instruction has finished executing |
| Exception flag + cause | raised only when this entry reaches the head |
| Branch info | predicted direction/target, RAT checkpoint id, mispredict flag |
(In a ROB-based design, a **value** field is added.)

**A circular buffer with head and tail pointers:**
```
   head (oldest: next to commit)                     tail (next free entry: newest)
     ▼                                                 ▼
 ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
 │  I1  │  I2  │  I3  │  I4  │  I5  │  I6  │ free │ free │ free │
 │ wait │ done │ done │ done │ exec │ wait │      │      │      │
 └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
 I1 is not done → NOTHING commits this cycle, even though I2–I4 are done.
```
- **Dispatch** writes new entries at the tail (in program order).
- **Commit**, each cycle: look at the head; if it's done with no exception, retire it (update retirement RAT, free old register; a store is released to the cache). Repeat for up to W entries.
- **ROB full** ⇒ dispatch stops ⇒ the front end stalls. A long cache miss at the head is the usual cause (N7).

**Exception at the head:**
1. Flush the **entire** ROB (everything younger than the faulting instruction).
2. Copy the **retirement RAT** into the speculative RAT, and rebuild the free list, so the state is exactly "everything before the faulting instruction done, nothing after".
3. Save the PC and cause, jump to the handler. This is how OoO cores give **precise exceptions**.
Wrong-path instructions that faulted (e.g., a page fault after a mispredicted branch) are squashed **before reaching the head**, so their exceptions never happen architecturally.

**Branch misprediction, two recovery styles:**
| Style | How | Trade-off |
|---|---|---|
| Recover at commit | wait until the branch reaches the head, then flush everything younger | simple, but adds the waiting time to the penalty |
| **Recover at execute** (modern) | as soon as the branch resolves wrong: flush only entries **younger** than the branch, restore the RAT **checkpoint**, redirect fetch | faster; needs checkpoints (limited number in flight) |

### 2.9 Issue queue: wakeup and select
Each cycle:
1. **Wakeup:** completing instructions broadcast their destination register tag. Every waiting entry compares its source tags (a CAM search) and marks matching sources ready.
2. **Select:** among entries with all sources ready, pick up to N per execution port (usually **oldest first**).
For **back-to-back** dependent instructions (e.g., `add` then a dependent `sub`, each 1 cycle), wakeup + select must complete in **one cycle**. This loop is one of the tightest timing paths in the core.
**Load speculation:** the scheduler wakes up a load's dependents *assuming an L1 hit*. If the load misses, those dependents are cancelled and **replayed** later.

### 2.10 Tomasulo's algorithm (IBM 360/91, 1967), with a worked example
```
         Instruction queue (in order)
                │ ISSUE: allocate RS, read ready operands or record producer TAG
   ┌────────────┼──────────────┐
   ▼            ▼              ▼
 [RS: Add1..3] [RS: Mul1..2]  [Load buffers]   ← reservation stations hold operands or tags
   │  EXECUTE when both operands present     │
   ▼                                          ▼
 FP adders    FP multipliers           Memory
   └──────────────┬──────────────────────────┘
          WRITE RESULT on Common Data Bus (CDB): (tag, value) broadcast
          → all RS waiting on that tag capture the value; register file updated
            only if its "Qi" still points to that tag
```
**Three steps per instruction:**
1. **Issue** (in order): if a reservation station (RS) of the right type is free, put the instruction there. For each source: if the value is available, copy it into the RS (**Vj/Vk**); otherwise record the **tag** of the RS that will produce it (**Qj/Qk**). Mark the destination register as "will be written by this RS" (**Qi**).
2. **Execute:** when both operands are present, execute.
3. **Write result:** broadcast (tag, value) on the **CDB**. Every RS waiting on that tag grabs the value. The register file is updated only if its Qi still equals that tag.

**RS fields:** Busy · Op · **Vj, Vk** (operand values) · **Qj, Qk** (tags of RSs producing operands not yet available; empty = value present) · (A = address, for loads/stores).
**Register status table:** for each register, **Qi** = tag of the RS that will write it (empty = register file value is current).

**Worked example** (H&P classic code). Assumptions: load 2 cycles, add/sub 2, multiply 10, divide 40; 2 load buffers, 3 add RSs, 2 multiply RSs; one CDB; an operand can be used the cycle after it's written.
```
I1: L.D   F6, 32(R2)
I2: L.D   F2, 44(R3)
I3: MUL.D F0, F2, F4
I4: SUB.D F8, F2, F6
I5: DIV.D F10, F0, F6
I6: ADD.D F6, F8, F2
```
| Instr | Issue | Execute | Write result (CDB) | Waits for |
|---|---|---|---|---|
| I1 L.D F6 | 1 | 2–3 | 4 | — |
| I2 L.D F2 | 2 | 3–4 | 5 | — |
| I3 MUL.D F0, F2, F4 | 3 | 6–15 | 16 | F2 (I2, cycle 5) |
| I4 SUB.D F8, F2, F6 | 4 | 6–7 | 8 | F2 (cycle 5), F6 (cycle 4) |
| I5 DIV.D F10, F0, F6 | 5 | 17–56 | 57 | F0 (I3, cycle 16) |
| I6 ADD.D F6, F8, F2 | 6 | 9–10 | 11 | F8 (I4, cycle 8) |

**Snapshot at the end of cycle 6:**
Reservation stations:
| RS | Busy | Op | Vj | Vk | Qj | Qk | Status |
|---|---|---|---|---|---|---|---|
| Load1, Load2 | no | | | | | | both loads finished (wrote at 4 and 5) |
| Add1 | yes | SUB | Mem[44+R3] (F2) | Mem[32+R2] (F6) | | | executing |
| Add2 | yes | ADD | | Mem[44+R3] (F2) | **Add1** | | waiting for SUB's result (F8) |
| Add3 | no | | | | | | |
| Mult1 | yes | MUL | Mem[44+R3] (F2) | Regs[F4] | | | executing |
| Mult2 | yes | DIV | | Mem[32+R2] (F6) | **Mult1** | | waiting for MUL's result (F0) |

Register status:
| | F0 | F2 | F4 | F6 | F8 | F10 |
|---|---|---|---|---|---|---|
| Qi | Mult1 | — | — | **Add2** | Add1 | Mult2 |

**What this example shows:**
1. **Out-of-order execution:** SUB (I4) starts at cycle 6 with MUL; ADD (I6) finishes at cycle 11, long before MUL (16) and DIV (57).
2. **WAR eliminated:** DIV reads F6 and the younger ADD writes F6. DIV **copied F6's value into its RS** at issue (Vk), so ADD can overwrite F6 at cycle 11 without harm.
3. **WAW handled:** F6's Qi points to **Add2**, the newest writer. If an older producer of F6 broadcast later, the register file would ignore it (Qi doesn't match), and only RSs waiting on that tag would use it.
4. **Forwarding built in:** waiting RSs grab values straight off the CDB; nobody waits for the register file.
5. **Problem: imprecise exceptions.** ADD has already written F6 at cycle 11. If DIV (older) divides by zero at cycle 56, the state isn't "everything before DIV done, nothing after". Adding a **ROB** with in-order commit fixes this (H&P's "speculative Tomasulo"). That's the ancestor of every modern OoO core.

### 2.11 Scoreboarding vs Tomasulo vs modern OoO
| | Scoreboard (CDC 6600, 1964) | Tomasulo (IBM 360/91, 1967) | Modern OoO (ROB + PRF) |
|---|---|---|---|
| Hazard tracking | one central scoreboard | distributed in reservation stations | rename + scheduler + ROB |
| RAW | wait; read operands from register file when ready | wait for tag on CDB, get value directly | wakeup on tag broadcast + bypass network |
| WAR | **stall** the write until earlier reads are done | **eliminated** (values copied into RS) | **eliminated** (renaming) |
| WAW | **stall** issue | **eliminated** (register status keeps newest tag) | **eliminated** (renaming) |
| Forwarding | no (through register file) | yes (CDB broadcast) | yes (bypass network) |
| Precise exceptions | no | no | **yes** (ROB) |
| Speculation past branches | no | no | **yes** |
| Bottleneck | stalls on WAR/WAW, register file ports | single CDB (1 result/cycle), associative RS lookup | wakeup/select timing, PRF ports, power |

### 2.12 Memory disambiguation and the load/store queue
Registers are renamed, but **memory addresses can't be renamed**, and an address is known only after it's computed.
```
sw  x5, 0(x6)     # store: x6 comes from a slow instruction → address unknown for a while
lw  x7, 0(x8)     # load: x8 is ready → can it execute before the store?
```
- If the addresses differ: the load can go early. ✅
- If they're the same: the load must get `x5` from the store. If it went early, it read stale memory. ❌

| Approach | How | Cost |
|---|---|---|
| Conservative | load waits until all older store addresses are known | loses ILP |
| **Speculative (modern)** | execute the load early; when the store address resolves, the store searches the **load queue** for younger loads to the same address that already executed ⇒ **memory-ordering violation** ⇒ flush the load and everything after it, re-execute | rare flushes |
| + **Memory dependence predictor** | e.g., **store sets**: remembers which loads conflicted with which stores before, and makes only those loads wait | fewer flushes |

- **Store-to-load forwarding:** if a load's address matches an older in-flight store in the **store queue**, take the data from the store queue (~4–5 cycles) instead of the cache. Partial overlaps (e.g., an 8-byte load from two 4-byte stores) may fail to forward ⇒ stall ~10+ cycles [HFT].
- **Stores write the cache only after commit.** Stores can't be undone once in memory (another core could see them), so they wait in the store queue/buffer until they are non-speculative.

### 2.13 Speculation: what's safe, and Spectre vs Meltdown
**Speculative instruction:** one that might not be on the correct path (after an unresolved predicted branch) or that might be cancelled (after an older instruction that might fault). The core runs them anyway, but keeps their effects **invisible** until commit.
| Can be done speculatively | Only at commit (or once non-speculative) |
|---|---|
| Compute into physical registers | Update the **architectural** (retirement) register state |
| Read the caches (loads) | **Write** the cache/memory (stores) |
| Allocate ROB/RS/LSQ entries | Raise exceptions |
| Update branch history (checkpointed, repaired on mispredict) | Access **MMIO / device memory** (reading a UART FIFO pops data; it can't be undone) [TI] |
Device memory is marked **non-speculative** in the page tables / MPU attributes, so the core never touches it on a wrong path.

**The catch:** squashing restores registers, but **not the cache**. A wrong-path load leaves a footprint (which line was brought in), and an attacker can measure that by timing. That's the root of transient-execution attacks:
| | **Spectre v1** | **Meltdown** |
|---|---|---|
| Trigger | branch **misprediction** (e.g., a bounds check predicted "in bounds") | a load that **faults** (user mode reading kernel memory), whose data is forwarded to dependents **before** the fault is taken at commit |
| What leaks | data the victim process can legally address (out-of-bounds within its own address space) | **kernel memory** from user space |
| Affected | almost all speculative CPUs (Intel, AMD, ARM, Apple…) | mainly Intel (also some ARM and IBM POWER cores); AMD not affected |
| Mitigation | `lfence`/speculation barriers, index masking, compiler fixes | **KPTI** (kernel page tables unmapped from user space), fixed in later hardware (don't forward faulting loads' data) |
See also [05 §2.9](05-Branch-Prediction.md).

### 2.14 Why OoO helps with cache misses, and its limit
While a load misses (say 300 cycles), the core keeps fetching and executing independent younger instructions until the **ROB fills**. A 512-entry ROB at 4 instructions/cycle fills in ~128 cycles. So OoO can **overlap independent misses** (memory-level parallelism, MLP), but it can't hide a single 300-cycle miss completely (N7). **Dependent misses** (pointer chasing: each address comes from the previous load) get no help at all (N6).

Modern window sizes (approx.): Intel Golden Cove ROB 512; AMD Zen 4 ROB 320; Apple Firestorm/M-series ~600+; issue width 6–10.

### 2.15 Superscalar costs (why not 32-wide?)
- Rename dependency checks across W instructions: O(W²).
- Wakeup/select and the bypass network: O(W²) wires.
- Register file ports grow with W ⇒ area/energy ⇒ often clustered designs.
- Programs rarely have more than 4–8 independent instructions per cycle on the critical path.
- Power: OoO structures (CAMs, wide RAMs) burn energy ⇒ efficiency cores are narrow and in-order (Cortex-A520), and **GPUs are in-order**, using thread-level parallelism instead ([12](12-GPU-Architecture.md)).
- Branch mispredicts throw away a bigger window: a wider, deeper machine loses more work per mispredict.

### 2.16 Software consequences [HFT][DESHAW][NVIDIA]
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

---

## 3. Worked Numericals

**N1. Chains needed to saturate units.** FMA latency 4 cycles, 2 FMA units/cycle ⇒ independent accumulators needed = latency × throughput = **8** (with AVX-512, each is a vector ⇒ 8 × 16 floats in flight).

**N2. ROB sizing via Little's law.** To keep issuing at IPC 4 across an L3 hit of 50 cycles: in-flight instructions = 4 × 50 = **200 ROB entries** (plus pipeline depth). For a 300-cycle DRAM miss: 1200, which is impractical. So DRAM misses still stall, unless the miss overlaps with other misses.

**N3. Unrolling speedup.** From the table in 2.2: 9 → 3.5 cycles/element = **2.57×** speedup with zero hardware change.

**N4. Physical registers needed.** 32 architectural registers + a 192-entry ROB where every instruction writes a register ⇒ **32 + 192 = 224** (each in-flight writer needs a fresh destination; committed state needs 32).

**N5. VLIW slot utilisation.** A 4-slot VLIW runs a loop body of 10 operations whose dependence graph has a critical path of 5 operations (1-cycle latency). Minimum cycles = max(⌈10/4⌉, 5) = **5** ⇒ utilisation 10/20 = **50%** (NOPs fill the rest: code bloat in classic VLIW).

**N6. Independent vs dependent misses (MLP).** Miss latency 200 cycles, everything else 1 cycle.
- **Independent:** `lw x1, 0(x2); lw x3, 0(x4); add x5, x1, x3`. OoO issues both loads back to back (cycles 2 and 3); data returns at ~202 and ~203; add at ~204 ⇒ **≈ 204 cycles**. A **blocking** in-order core (stalls on every miss) ⇒ ≈ 400+.
- **Dependent (pointer chase):** `lw x1, 0(x2); lw x3, 0(x1)`. The second address comes from the first load ⇒ **≈ 400 cycles on any core**.
- Nuance: an in-order core with a **non-blocking cache** that stalls only when a value is *used* would also overlap the two independent loads above. OoO's real advantage is finding independent misses **much further ahead** (up to the ROB size), even past stalled instructions.

**N7. How much of a miss can the ROB hide?** ROB = 224 entries, dispatch 4/cycle, a load at the ROB head misses for 300 cycles, and everything after it is independent.
- The other 223 entries fill in 223/4 ≈ **56 cycles**; then the ROB is full and dispatch stops.
- The core stalls for the remaining 300 − 56 ≈ **244 cycles**. Only ~**19%** of the miss is hidden.
- If another independent miss appears within those 223 instructions, it overlaps with the first one, which is the real benefit.

**N8. In-order vs OoO timing.** Same model as 2.5 (dispatch 1/cycle, commit up to 4/cycle). Load miss 6 cycles, others 1.
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
- **Q: Walk me through renaming `I1: x1 = …; I2: … = x1; I3: x1 = …`.** Use the 4 rules in 2.6; show I3 gets a new physical register so it doesn't wait for I1/I2.
- **Q: When is a physical register freed?** When the next instruction that writes the same architectural register commits (it frees the old mapping it saved).
- **Q: Explain Tomasulo's algorithm.** Issue (RS + tags), execute when operands ready, write result on CDB; renaming via tags; values copied into RS remove WAR (2.10).
- **Q: Scoreboarding vs Tomasulo?** Scoreboard stalls on WAR/WAW and has no forwarding; Tomasulo eliminates them via tags and forwards over the CDB (2.11).
- **Q [TI][QCOM]: VLIW vs superscalar? Why do DSPs use VLIW?** Regular, predictable loops (filters, FFT) let the compiler schedule perfectly ⇒ high throughput at low power.
- **Q: What is loop unrolling and its downsides?** (2.2.)

### Advanced / killer follow-ups
- **Q [APPLE]: How does a load execute before an earlier store with an unknown address?** Speculative disambiguation + load-queue check when the store address resolves + memory dependence predictor + flush/replay (2.12).
- **Q: What is store-to-load forwarding and when does it fail?** Load reads from the store queue; fails on partial overlap/misalignment ⇒ wait for the store to commit.
- **Q: What happens on a branch mispredict in an OoO core?** Flush younger ROB entries, free their new physical registers, restore the RAT checkpoint, redirect fetch; modern cores do this at execute, not at commit (2.8).
- **Q: How are precise exceptions achieved in an OoO core?** Exceptions are recorded in the ROB entry and taken only at the head; everything younger is flushed and the retirement RAT restored.
- **Q: ROB-based renaming vs unified PRF?** (2.7.) PRF avoids copying values at commit (important for wide vectors); P6 stored values in the ROB.
- **Q: Why doesn't increasing ROB size indefinitely help?** Branch mispredicts (window full of wrong-path work), dependent misses, and the power/area/timing of wakeup-select and the PRF.
- **Q: Why can't MMIO reads be executed speculatively?** Reading a device register can have side effects (popping a FIFO, clearing a flag); device memory is marked non-speculative.
- **Q: Spectre vs Meltdown?** (2.13.) Both exploit cache footprints of squashed instructions; Spectre via mispredicted branches, Meltdown via data forwarded from a faulting load.
- **Q [NVIDIA]: Why are GPU cores in-order while CPUs are OoO?** GPUs target throughput: thousands of threads hide latency (TLP), which is far cheaper per FLOP than OoO hardware built for single-thread latency.
- **Q [HFT]: Your summation loop runs at 1/8 of peak FLOPs. Why, and fix?** A single accumulator forms a dependency chain; use multiple accumulators / SIMD; mind FP reassociation.

---

## 5. Common Traps
- Saying renaming removes RAW. It doesn't; only WAR and WAW.
- Saying OoO commits out of order. It **executes** out of order and **commits in order**.
- Freeing a physical register when its reader executes. It's freed when the **next writer of that architectural register commits**.
- Thinking OoO hides all memory latency (N7) or helps pointer chasing (N6).
- Forgetting that stores update memory only at commit.
- Thinking squashing undoes everything. It undoes registers, but **not cache state** (Spectre/Meltdown).
- Confusing Tomasulo's Q fields (tags) with V fields (values).

---

## 6. Self-Check Quiz
1. In the 2.5 walkthrough, why can't I3 commit at cycle 5, even though it finished at cycle 4?
2. Rename this code. Initial RAT: `x1→P1, x2→P2, x3→P3`, free list P4, P5, P6.
   `I1: add x1, x2, x3` · `I2: sub x2, x1, x1` · `I3: add x1, x2, x1`
3. In question 2, which physical register is freed when I3 commits?
4. Which two hazards does Tomasulo eliminate that a scoreboard only stalls on, and how?
5. A 256-entry ROB, dispatch 4/cycle, a 400-cycle miss at the head. Roughly how many cycles of the miss are hidden?
6. Why must a store wait until commit before writing the cache, while a load can read the cache speculatively?

<details><summary>Answers</summary>

1. Commit is in order. I1 (older) is still waiting for memory at the ROB head, so nothing behind it can commit.
2. I1: `add P4, P2, P3` (x1→P4, saves old P1). I2: `sub P5, P4, P4` (x2→P5, saves old P2). I3: `add P6, P5, P4` (x1→P6, saves old P4). Final RAT: x1→P6, x2→P5, x3→P3.
3. **P4**: the old mapping of x1 saved in I3's ROB entry. (I1 frees P1 at its commit, I2 frees P2.)
4. **WAR and WAW.** WAR: source values are copied into the reservation station at issue, so a later write can't clobber them. WAW: the register status table always points to the newest producer's tag, so an older result never overwrites a newer one.
5. The other 255 entries fill in about 255/4 ≈ **64 cycles**; the remaining ~336 cycles stall (≈ 16% hidden).
6. A store to memory can't be undone, since other cores or devices may observe it. A speculative load changes no architectural state (only cache contents, the Spectre caveat), so it's simply discarded on a squash.
</details>

---

## 7. Revision Summary
| | |
|---|---|
| ILP limits | RAW (dataflow, can't remove), WAR/WAW (renaming), control (prediction), memory (disambiguation), resources |
| Static ILP | unrolling, scheduling, software pipelining, VLIW (TI C6000, Hexagon) |
| OoO shape | in-order front (fetch → rename → dispatch) · OoO execute · in-order commit |
| Renaming | sources via RAT; new dest from free list; save old mapping in ROB; free old mapping when this instruction commits |
| Squash | free new registers, restore RAT checkpoint, redirect fetch |
| Renaming styles | ROB-based values (P6) vs unified PRF (R10000, modern) |
| ROB | circular buffer; commit from head when done; exceptions taken at head (precise) |
| Wakeup/select | tag broadcast + oldest-ready selection; 1-cycle loop; load-hit speculation + replay |
| Tomasulo | Issue (Vj/Vk or Qj/Qk tags) → Execute → CDB write; kills WAR/WAW; imprecise without ROB |
| Scoreboard | stalls on WAR/WAW, no forwarding |
| Memory | speculative load bypass, load-queue check, store sets, store-to-load forwarding, stores at commit |
| Speculation | registers rolled back, cache not ⇒ Spectre (mispredict) / Meltdown (faulting load) |
| Miss hiding | ROB fill time = ROB / width; overlaps independent misses (MLP), not dependent |
| Code | break dependency chains (latency × throughput accumulators) |
