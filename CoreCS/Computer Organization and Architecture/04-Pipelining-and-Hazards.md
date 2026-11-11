# 04. Pipelining & Hazards

> **Asked at:** Every hardware/architecture interview. [NVIDIA] [APPLE] [QCOM] hazards, forwarding, deep vs shallow pipelines · [TI] pipeline timing numericals, DSP delay slots · [HFT] why mispredicts cost ~15–20 cycles · [DESHAW] stall counting for a code snippet.
> **Prerequisites:** [02 Performance](02-Performance-Metrics-and-Laws.md), [03 ISA](03-ISA-and-Instruction-Formats.md).

---

## 1. Intuition

### 1.1 The laundry analogy
You have 4 loads of laundry. Each load goes through 3 steps: **wash (30 min) → dry (40 min) → fold (20 min)** = 90 min per load.

**Without pipelining**, you finish one load completely before starting the next:
```
Time (min): 0        90       180      270      360
Load 1:     [W][D ][F]
Load 2:              [W][D ][F]
Load 3:                       [W][D ][F]
Load 4:                                [W][D ][F]          → 360 min
```
**With pipelining**, you start washing load 2 as soon as load 1 moves to the dryer:
```
Time (min): 0   30  70  110 150 190 210
Load 1:     [W ][D  ][F ]
Load 2:         [W ]   [D  ][F ]               (waits for dryer: dryer is the slowest step)
Load 3:                [W ]    [D  ][F ]
Load 4:                       ...   [D  ][F ]   → ≈ 90 + 3 × 40 = 210 min
```
Three lessons that apply directly to CPUs:
1. **Each load still takes ≥ 90 min** (latency is *not* reduced). But **one load finishes every 40 min** (throughput goes up).
2. **The slowest step (dryer) sets the rhythm.** In a CPU, the slowest stage sets the clock period. Stages should be balanced.
3. **Things can go wrong.** If you have only one machine that both washes and dries, two loads can't use it at once (*structural hazard*). If load 2 needs a shirt from load 1 that isn't folded yet (*data hazard*). If you don't know which load comes next until load 1 is inspected (*control hazard*).

### 1.2 Vocabulary used in this note
| Term | Meaning |
|---|---|
| **Stage** | One step of instruction processing (IF, ID, EX, MEM, WB) |
| **Pipeline register (latch)** | Storage between two stages; holds one instruction's data and control bits |
| **Latency** | Time for *one* instruction to go from start to finish |
| **Throughput** | Instructions finished per unit time |
| **CPI** | Cycles per instruction; ideal pipeline CPI = 1 |
| **Hazard** | A situation that prevents the next instruction from proceeding in the next cycle |
| **Stall / bubble** | A cycle where a stage does no useful work (a NOP flows through) |
| **Flush** | Throwing away instructions already in the pipeline (wrong-path after a branch, or after an exception) |
| **Forwarding / bypassing** | Sending a result directly from a later stage to an earlier one, without waiting for it to be written to the register file |

---

## 2. Deep Dive

### 2.1 Why pipeline? Single-cycle vs multi-cycle vs pipelined
Typical step delays (Patterson & Hennessy example): instruction fetch 200 ps, decode + register read 100 ps, ALU 200 ps, data memory 200 ps, register write 100 ps.

| Instruction | IF | ID (decode + reg read) | ALU | Mem | Reg write | **Total** |
|---|---|---|---|---|---|---|
| `lw` | 200 | 100 | 200 | 200 | 100 | **800 ps** |
| `sw` | 200 | 100 | 200 | 200 | — | 700 ps |
| R-type (`add`) | 200 | 100 | 200 | — | 100 | 600 ps |
| `beq` | 200 | 100 | 200 | — | — | 500 ps |

> **Where is instruction decode?** In the **ID** stage, not IF. Decoding (control unit reads the opcode/funct bits) and register reading happen **in parallel** within the same ~100 ps. This works because RISC-V always places `rs1`/`rs2` in fixed bit positions, so the register file can start reading before decode finishes. If an instruction doesn't use `rs2` (e.g., `addi`, `lw`), the extra value is simply ignored. The stage is labelled by its slower part (register read), which is why some textbooks write just "Reg read".

| Design | How it works | Clock period | CPI | Problem |
|---|---|---|---|---|
| **Single-cycle** | every instruction finishes in one long cycle | 800 ps (the slowest instruction, `lw`) | 1 | `add` only needs 600 ps but still pays 800 ps. Hardware sits idle most of the cycle |
| **Multi-cycle** | each instruction uses as many short cycles as it needs | 200 ps (slowest *step*) | 3–5 (depends on instruction) | only **one** instruction is in progress at a time |
| **Pipelined** | each short cycle, a *new* instruction starts; up to 5 in flight | 200 ps (+ latch overhead) | ideally **1** | hazards |

Why the pipelined version wins: it combines the **short clock** of multi-cycle with the **CPI ≈ 1** of single-cycle.

### 2.2 The classic 5-stage RISC pipeline: what each stage does
```
 ┌────┐  ┌──────┐  ┌────┐  ┌──────┐  ┌────┐  ┌───────┐  ┌─────┐  ┌──────┐  ┌────┐
 │ IF │─►│IF/ID │─►│ ID │─►│ID/EX │─►│ EX │─►│EX/MEM │─►│ MEM │─►│MEM/WB│─►│ WB │
 └────┘  └──────┘  └────┘  └──────┘  └────┘  └───────┘  └─────┘  └──────┘  └────┘
 I-cache            Register file      ALU                D-cache               Register file
 PC += 4            (2 read ports)                                              (1 write port)
```
| Stage | Hardware used | `lw x1, 8(x2)` | `sw x1, 8(x2)` | `add x3, x1, x2` | `beq x1, x2, L` |
|---|---|---|---|---|---|
| **IF** (fetch) | PC, I-cache, PC adder | fetch instr, PC ← PC+4 | same | same | same |
| **ID** (decode) | control unit, register file read, sign-extend | read x2, sign-extend 8 | read x2 **and** x1 | read x1, x2 | read x1, x2; compute target PC+offset |
| **EX** (execute) | ALU | address = x2 + 8 | address = x2 + 8 | x1 + x2 | compare x1, x2 (subtract, check zero) → decide taken/not taken |
| **MEM** (memory) | D-cache | read memory[address] | write x1 into memory[address] | — (passes through) | (in some designs: update PC here) |
| **WB** (write back) | register file write | x1 ← loaded value | — | x3 ← sum | — |

Key observations:
- **Each hardware unit is used by exactly one stage.** This is what lets 5 different instructions use 5 different units in the same cycle.
- Every instruction goes through all 5 stages (even if it does nothing in some), so every instruction takes the same path. This keeps control simple.
- RISC ISA design makes this easy: fixed-length instructions (IF is simple), register fields in fixed positions (ID can read registers while decoding), only loads/stores touch memory (one MEM stage is enough). See [03](03-ISA-and-Instruction-Formats.md).

### 2.3 Pipeline registers: what they carry
Because 5 different instructions are in flight, each instruction's information must **travel with it** through the pipeline. The registers between stages hold that information.

| Pipeline register | Contents |
|---|---|
| **IF/ID** | the fetched instruction bits, PC+4 |
| **ID/EX** | values read from rs1 and rs2, sign-extended immediate, rd number, rs1/rs2 numbers (for forwarding), PC, **all control signals for EX, MEM, WB** |
| **EX/MEM** | ALU result (or address), value to store (for `sw`), rd, branch target & zero flag, control signals for MEM and WB |
| **MEM/WB** | data loaded from memory, ALU result, rd, control signals for WB |

**Control signals travel too.** The control unit decodes the instruction once, in ID. The signals are then carried along and "used up" stage by stage:
```
 ID/EX carries:  [EX: ALUOp, ALUSrc] [MEM: MemRead, MemWrite, Branch] [WB: RegWrite, MemToReg]
 EX/MEM carries:                     [MEM: MemRead, MemWrite, Branch] [WB: RegWrite, MemToReg]
 MEM/WB carries:                                                      [WB: RegWrite, MemToReg]
```
Why does `rd` have to travel all the way to WB? Because by the time `lw x1` reaches WB, the instruction in ID is a completely different one. The register number must come from the `lw`'s own MEM/WB register, not from the instruction currently being decoded. (A classic bug in student designs.)

**A bubble (NOP) = an instruction whose control signals are all 0**: it writes no register and no memory, so it changes nothing as it flows through.

### 2.4 Reading and drawing pipeline diagrams
Rows = instructions (in program order), columns = clock cycles. Each cell shows which stage the instruction is in during that cycle.
```
Cycle:  1   2   3   4   5   6   7   8
I1      IF  ID  EX  MEM WB
I2          IF  ID  EX  MEM WB
I3              IF  ID  EX  MEM WB
I4                  IF  ID  EX  MEM WB
```
Read a **column** to see what the hardware does in one cycle. In cycle 5: I1 is writing back, I2 is accessing memory, I3 is in the ALU, I4 is reading registers. Four different units are busy at once.

Notation used in this note:
| Symbol | Meaning |
|---|---|
| `ID*` / `IF*` | the instruction is **stalled**, stuck in that stage for this cycle |
| `--` or `bubble` | a NOP is flowing through that stage |
| `flush` | the instruction is cancelled (turned into a NOP) |

The first 4 cycles are the **fill** (pipeline not yet full). The last 4 cycles of a program are the **drain**.

### 2.5 Pipeline timing formulas (with derivation)
Let **k** = number of stages, **n** = number of instructions, **τ** = clock period.

**Clock period:**
```
τ = max(stage delays) + latch overhead
```
Latch overhead = setup time + clock-to-Q delay + clock skew of the pipeline register. It is paid in **every** stage, every cycle.

**Total time, pipelined:** the first instruction needs k cycles to get through. After that, one instruction finishes every cycle, so the remaining (n − 1) instructions need (n − 1) more cycles.
```
Time_pipelined   = (k + n − 1) × τ
Time_unpipelined = n × (sum of all stage delays)
```
**Speedup:**
```
Speedup = Time_unpipelined / Time_pipelined
        = n × T_total / ((k + n − 1) × τ)

For large n:   Speedup → T_total / τ
Ideal case (balanced stages, zero latch overhead):  τ = T_total / k  ⇒  Speedup → k
```
**With stalls** (Hennessy & Patterson form, balanced stages, unpipelined CPI = 1):
```
Speedup = Pipeline depth / (1 + stall cycles per instruction)
CPI_pipelined = 1 + structural stalls + data stalls + control stalls     (per instruction)
```
Why real speedup is less than k:
1. **Stage imbalance:** the clock follows the slowest stage; faster stages wait.
2. **Latch overhead:** added to every stage.
3. **Hazards:** stalls and flushes raise CPI above 1.
4. **Fill and drain:** matter only when n is small.

### 2.6 Hazards: overview
A **hazard** is anything that stops the next instruction from executing in its scheduled cycle.

| Hazard | Cause | Example | Fixes |
|---|---|---|---|
| **Structural** | two instructions need the **same hardware** in the same cycle | one shared memory for IF and MEM | duplicate the resource, pipeline the resource, or stall |
| **Data** | an instruction needs a **result not yet produced** | `add x1,…` followed by `sub …, x1, …` | forwarding, stall, compiler scheduling |
| **Control** | the **next PC isn't known** yet | `beq`, `jalr` | stall, prediction, delayed branch, predication |

### 2.7 Structural hazards in detail
**Example: a single memory for instructions and data.** In cycle 4, I1 (`lw`) is in MEM reading data, while I4 is in IF fetching an instruction. Both need the memory:
```
Cycle:          1   2   3   4   5   6   7   8
lw  x1,0(x2)    IF  ID  EX  MEM WB                 ← uses memory in cycle 4
I2                  IF  ID  EX  MEM WB
I3                      IF  ID  EX  MEM WB
I4                          IF* IF  ID  EX  MEM WB ← also needs memory in cycle 4 → must wait 1 cycle
```
Fix: **separate instruction and data caches** (L1-I and L1-D), the "modified Harvard" design all modern CPUs use.

Other structural hazards and how designs avoid them:
| Resource | Conflict | Solution |
|---|---|---|
| Register file | ID reads 2 registers while WB writes 1 in the same cycle | 2 read ports + 1 write port; **write in first half of cycle, read in second half** |
| Memory | IF and MEM in the same cycle | split I-cache / D-cache |
| Non-pipelined divider | two divides back-to-back | pipeline the unit, add another unit, or stall the second divide |
| Register-file write port | a long-latency FP op and a short integer op finish in the same cycle | stall one, or add write ports (see 2.14) |

Structural hazards are usually removed **by design**, because extra hardware is cheaper than lost cycles. Designers sometimes accept them for rare, expensive units (e.g., a single divider).

> **Can a register read and a memory read happen in the same cycle?** **Yes.** The register file is an SRAM array *inside the core*, with its own dedicated read/write ports and wires; it doesn't use any memory bus. In cycle 4 of the diagram above, four units work at once: IF reads the **I-cache**, ID reads the **register file**, EX uses the **ALU**, MEM reads the **D-cache**. Each has its own hardware and its own wiring (point-to-point datapath with muxes, not one shared bus), so there's no conflict. Conflicts arise only when two stages need the **same** unit: IF + MEM sharing one memory (fixed by split I/D caches), or ID reading + WB writing the register file (fixed by separate read/write ports and the write-first-half/read-second-half trick). The external memory bus to DRAM is used only on a cache miss.
>
> **Exception: single-bus CPU organisation** (Hamacher, GATE-style questions). There, registers, ALU, and MDR/MAR all hang on **one internal bus**, and only one value can be placed on it per cycle. So moving a register's value and moving memory data (via MDR) *do* compete for the bus. Two-bus and three-bus organisations reduce this. Pipelined datapaths avoid it entirely with dedicated paths.

> **But what if the I-cache and D-cache both miss in the same cycle?** Both then need the **shared, unified L2**. Split L1s don't remove the structural hazard; they **move it to L2**, where it is **rare**, because both must miss in the same cycle. It's handled like this:
> 1. **Separate request paths + an arbiter at L2.** The L1I and L1D each have their own link to L2 (not one shared bus). If both requests arrive together, an **arbiter** picks one first (fixed priority or round-robin; in a simple in-order pipeline the D-miss belongs to the older instruction, so it's often served first). The other waits in a **request queue**, usually for 1 cycle.
> 2. **The L2 is pipelined and banked.** It can *accept* a new request every cycle (or several per cycle across banks) while earlier ones are still in progress. So both misses are **in flight at the same time**, offset by about 1 cycle, rather than one waiting for the other to fully finish.
> 3. **The pipeline stalls meanwhile.** In a simple in-order 5-stage pipeline, any L1 miss stalls the pipeline until the data returns. A 1-cycle arbitration delay is tiny next to a ~12-cycle L2 access. Out-of-order cores use **non-blocking caches** (MSHRs) to keep many misses outstanding ([06 §2.9](06-Cache-Memory.md)).
> 4. **Further down** (L3, DRAM), the same pattern repeats: queues, arbiters, banks and channels on an on-chip ring/mesh. Many requests are in flight, and each resource is granted to one request at a time.
>
> Key distinction: two requests usually can't **start** at the same resource in the same cycle, but they can **be in progress** at the same time (pipelining/overlap).

### 2.8 Data hazards: RAW, WAR, WAW
Named after the order of operations in **program order**: the second instruction does "X after the first instruction's Y".
```
I1: add x1, x2, x3
I2: sub x4, x1, x5     RAW on x1 with I1: I2 Reads x1 After I1 Writes it  (true dependence)
I3: and x2, x6, x7     WAR on x2 with I1: I3 Writes x2 After I1 Reads it  (anti-dependence)
I4: or  x1, x8, x9     WAW on x1 with I1: I4 Writes x1 After I1 Writes it (output dependence)
```
| Type | Also called | Real data flow? | Danger if violated |
|---|---|---|---|
| **RAW** (Read After Write) | true dependence | **yes**: I2 needs I1's value | I2 reads the old value of x1 |
| **WAR** (Write After Read) | anti-dependence | no, just reuse of a register *name* | I3 overwrites x2 before I1 has read it |
| **WAW** (Write After Write) | output dependence | no, just reuse of a register *name* | x1 ends up with I1's value instead of I4's |

**Why only RAW matters in the classic 5-stage pipeline:**
- All register **reads** happen early (ID), all register **writes** happen late (WB), and instructions move strictly **in order**.
- WAR: I3 writes x2 in its WB, which is always after I1's ID (I1 is older). I1 already read x2. Safe.
- WAW: I1's WB always happens before I4's WB (in order, same stage). Final value is I4's. Safe.

WAR and WAW become real problems only when instructions can **finish out of order**: multi-cycle units (2.14) or out-of-order cores. **Register renaming** removes them ([09](09-ILP-Superscalar-Out-of-Order.md)). RAW can never be removed, only made shorter (forwarding) or hidden (by doing other work meanwhile).

### 2.9 Handling RAW without forwarding: stall until write-back
Without forwarding, the consumer must read the register file **in or after the producer's WB cycle**. Thanks to the split-cycle register file (write first half, read second half), the consumer's ID may be in the *same* cycle as the producer's WB.

```
Cycle:            1   2   3   4   5   6   7   8
add x1,x2,x3      IF  ID  EX  MEM WB            ← x1 written in the first half of cycle 5
sub x4,x1,x5          IF  ID  ID* ID* EX  MEM WB ← stays in ID until cycle 5, reads x1 in second half
```
The `sub` would normally be in ID in cycle 3; it has to wait until cycle 5 ⇒ **2 stall cycles**.

**Stalls without forwarding, by distance** (distance 1 = the very next instruction):
| Distance from producer | Stalls (no forwarding) |
|---|---|
| 1 | 2 |
| 2 | 1 |
| 3 or more | 0 |
This is the same whether the producer is an ALU instruction or a load, because without forwarding everyone waits for WB.

### 2.10 Forwarding (bypassing) in detail
**Key observation:** the `add` result is computed at the **end of EX (cycle 3)**. The `sub` needs it at the **start of its EX (cycle 4)**. The value already exists; it's just sitting in the EX/MEM pipeline register, not yet in the register file. So route it directly to the ALU input.

```
                         ┌─────────────── forward from MEM/WB (producer 2 ahead) ───────────┐
                         │      ┌──────── forward from EX/MEM (producer 1 ahead) ──┐        │
                         ▼      ▼                                                  │        │
 ID/EX.rs1_value ──►  ┌──────────┐                                                 │        │
                      │ 3:1 MUX  │──► ALU input A ──► [ ALU ] ──► EX/MEM.result ───┘        │
                      └──────────┘   (ForwardA selects)                 │                   │
                                                                         ▼                   │
                                                                  MEM ──► MEM/WB.value ──────┘
 (an identical MUX, controlled by ForwardB, feeds ALU input B from rs2)
```
**Forwarding unit logic (for ALU input A; repeat for rs2 / ForwardB):**
```
// EX hazard: producer is the instruction 1 ahead (now in MEM stage)
if (EX/MEM.RegWrite and EX/MEM.rd != 0 and EX/MEM.rd == ID/EX.rs1)
        ForwardA = 10      // take EX/MEM.result

// MEM hazard: producer is 2 ahead (now in WB stage), and the 1-ahead instruction did NOT match
else if (MEM/WB.RegWrite and MEM/WB.rd != 0 and MEM/WB.rd == ID/EX.rs1)
        ForwardA = 01      // take MEM/WB value (loaded data or ALU result)

else    ForwardA = 00      // no hazard: use value read from register file
```
Why each condition:
| Condition | Reason |
|---|---|
| `RegWrite` | only instructions that actually write a register (not `sw`, `beq`) can be producers |
| `rd != 0` | x0 is hard-wired to 0; an instruction "writing" x0 must not forward a non-zero value |
| `rd == rs1` | the producer writes the exact register the consumer reads |
| EX/MEM checked **first** | if both stages match, the **more recent** value (1 ahead) is the correct one |

Example of why priority matters:
```
add x1, x1, x2     ← I1
add x1, x1, x3     ← I2 (needs I1's x1: EX/MEM forward)
add x1, x1, x4     ← I3: both I1 (in MEM/WB) and I2 (in EX/MEM) wrote x1 → must take I2's (newer) value
```

**Distance table with full forwarding:**
| Producer | Distance | Stalls | Value comes from |
|---|---|---|---|
| ALU instr | 1 | 0 | EX/MEM → EX |
| ALU instr | 2 | 0 | MEM/WB → EX |
| ALU instr | 3 | 0 | register file (write first half / read second half) |
| **Load** | **1** | **1** | MEM/WB → EX, after 1 stall (see 2.11) |
| Load | 2 | 0 | MEM/WB → EX |
| Load | 3 | 0 | register file |

**Store data:** `sw x1, 0(x3)` needs x1 only in MEM (as the data to write), not in EX. So `lw x1, …` immediately followed by `sw x1, …` can be handled **without a stall** with an extra **MEM/WB → MEM forwarding path**. Many textbook designs omit this path and stall instead; say which you assume.

### 2.11 The load-use hazard (forwarding can't fully fix it)
A load's value comes out of the **D-cache at the end of MEM**. The next instruction needs it at the **start of EX**, which is the *same* cycle as the load's MEM. The value doesn't exist yet. Forwarding can't send data backwards in time.

```
Without a stall (impossible):
Cycle:          1   2   3   4   5
lw  x1,0(x2)    IF  ID  EX  MEM WB          ← x1 available only at END of cycle 4
add x3,x1,x4        IF  ID  EX              ← needs x1 at START of cycle 4 ✗

With 1 stall (correct):
Cycle:          1   2   3   4   5   6   7   8   9
lw  x1,0(x2)    IF  ID  EX  MEM WB
add x3,x1,x4        IF  ID  ID* EX  MEM WB             ← 1 stall; then MEM/WB → EX forward in cycle 5
sub x5,x3,x6            IF  IF* ID  EX  MEM WB         ← also delayed by 1; gets x3 via EX/MEM → EX
sw  x5,8(x2)                    IF  ID  EX  MEM WB     ← gets x5 via EX/MEM → EX
```
4 instructions take 9 cycles instead of 8 ⇒ exactly 1 stall.

**Hazard detection unit (located in ID):**
```
if (ID/EX.MemRead and
    (ID/EX.rd == IF/ID.rs1 or ID/EX.rd == IF/ID.rs2))
        stall
```
In words: "the instruction currently in EX is a load, and the instruction currently in ID reads the register that load is writing."

**What "stall" means in hardware** (three actions, all in the same cycle):
1. **PCWrite = 0**: the PC is not updated, so IF fetches the same instruction again.
2. **IF/IDWrite = 0**: the IF/ID register keeps its contents, so the dependent instruction stays in ID.
3. **Zero the control signals going into ID/EX**: a bubble (NOP) goes into EX instead of the dependent instruction.
The next cycle, the load is in WB, the condition is false, and the dependent instruction proceeds with forwarding from MEM/WB.

**Compiler fix: instruction scheduling.** Put an independent instruction between the load and its use.
```c
// a = b + c;  d = e + f;
```
```asm
# Naive order: 2 load-use stalls
lw   t1, b
lw   t2, c
add  t3, t1, t2     # uses t2 right after its load → stall
sw   t3, a
lw   t4, e
lw   t5, f
add  t6, t4, t5     # uses t5 right after its load → stall
sw   t6, d

# Scheduled: 0 stalls (same instructions, different order)
lw   t1, b
lw   t2, c
lw   t4, e          # moved up: fills the slot after "lw t2"
add  t3, t1, t2     # t2 is now 2 instructions old → no stall
lw   t5, f
sw   t3, a          # fills the slot after "lw t5"
add  t6, t4, t5     # t5 is now 2 instructions old → no stall
sw   t6, d
```
Using different registers (t4, t5, t6 instead of reusing t1, t2, t3) is what allows the reordering. Reusing names would create WAR/WAW constraints. This is register renaming done by the compiler.

### 2.12 Control hazards in detail
**The problem:** IF must fetch a new instruction **every cycle**, but whether `beq` is taken (and where it goes) is only known in **EX**. By then, two more instructions have been fetched from PC+4 and PC+8.

**Branch resolved in EX, predict-not-taken, branch is taken:**
```
Cycle:               1   2   3   4   5   6   7   8
beq x1,x2,L (taken)  IF  ID  EX  MEM WB               ← outcome known at end of cycle 3
I+1 (wrong path)         IF  ID  flush                ← cancelled
I+2 (wrong path)             IF  flush                ← cancelled
L: (target)                      IF  ID  EX  MEM WB   ← fetched in cycle 4
```
Two cycles of work thrown away ⇒ **branch penalty = 2 cycles**. If the branch is *not* taken, the instructions fetched from PC+4, PC+8 were correct ⇒ **0 penalty**.

**Flushing** = turning the wrong-path instructions in IF/ID and ID/EX into NOPs (zero their control bits). They haven't written any register or memory yet (that happens only in MEM/WB), so cancelling them is safe.

**Reducing the penalty: resolve the branch in ID.** Add a comparator (equality check) and a target adder in ID:
```
Cycle:               1   2   3   4   5   6
beq x1,x2,L (taken)  IF  ID  EX  MEM WB         ← outcome known at end of cycle 2
I+1 (wrong path)         IF  flush
L: (target)                  IF  ID  EX  MEM WB
```
Penalty drops to **1 cycle**, but there's a cost:
- The branch now needs its operands in **ID**, one stage earlier. If the previous instruction computes x1 (ALU result at end of its EX), the branch must **stall 1 cycle**. If the previous instruction is a load producing x1, it must **stall 2 cycles**.
- Needs forwarding paths into ID and a comparator there, which may **lengthen the ID stage** and therefore the clock period.

**Jumps:** `jal` (target = PC + offset) is known in ID ⇒ 1-cycle penalty without prediction. `jalr` (target in a register) needs the register value ⇒ like a branch. Returns are predicted with a return address stack ([05](05-Branch-Prediction.md)).

**Strategies:**
| Strategy | How it works | Penalty paid | Notes |
|---|---|---|---|
| **Stall** | freeze fetch until the branch resolves | on **every** branch | simplest, slowest |
| **Predict not-taken** | keep fetching PC+4, PC+8; flush if taken | only on **taken** branches | simple; loops are mostly taken, so not great |
| **Predict taken** | fetch from target | needs target early (BTB) | good for loops |
| **Dynamic prediction + BTB** | predictor guesses direction, BTB gives target, both in IF | only on **mispredictions** | what all modern CPUs do; see [05](05-Branch-Prediction.md) |
| **Delayed branch** | ISA rule: the instruction after the branch **always executes** | compiler fills the slot | MIPS, SPARC; DSPs like TI C6000 have 5 branch delay slots |
| **Predication** (`cmov`, `csel`) | turn "if" into a data dependence; no branch at all | none, but both sides computed | good for short, unpredictable branches |

**Delayed branch example (MIPS-style, 1 delay slot):**
```asm
# Original                         # Scheduled into the delay slot
add  x1, x2, x3                    beq  x4, x5, L
beq  x4, x5, L                     add  x1, x2, x3     # delay slot: runs whether or not branch is taken
nop  (delay slot)                  ...
```
Legal because `add` doesn't affect the branch condition. If no safe instruction exists, the compiler puts a `nop`. Delayed branches made sense for a 5-stage pipeline with a 1-cycle penalty. On a 15-stage or superscalar pipeline, the penalty is many cycles but the ISA still promises only 1 slot. It becomes useless baggage, which is why RISC-V and ARMv8 don't have them.

**Cost formula:**
```
Branch stall cycles per instruction = branch frequency × misprediction rate × penalty
```
(For predict-not-taken, the "misprediction rate" is the fraction of branches that are taken.)
Modern CPUs: penalty ≈ **15–20 cycles**, because branches resolve after fetch, decode, rename, scheduling and execution. Even a 5% misprediction rate with 20% branches costs 0.2 × 0.05 × 17 ≈ **0.17 CPI**. That is huge when a 4-wide core's ideal CPI is 0.25.

> **Why 2 cycles in the 5-stage pipeline but ~17 in a modern CPU?** Same rule in both:
> ```
> Branch penalty ≈ (stage where the branch resolves) − (stage where fetch picks the next PC) [+ redirect cycles]
> ```
> - **5-stage:** fetch is stage 1, the branch resolves in EX = stage 3 ⇒ 3 − 1 = **2** wrong-path instructions (fetched in the 2 cycles before the outcome is known).
> - **Modern out-of-order core** (illustrative stage counts; real designs vary):
>   ```
>   Stage:  1  2  3 | 4  5 | 6  7  8 | 9  10 | 11 | 12 | 13  14 | 15 | 16–17
>           Fetch   | Queue| Decode  | Rename| Disp| Sched| Reg read| EX | Redirect
>           (predict + I-cache)  (x86: length decode, µops)  (RAT, ROB alloc)  (resolved here) (flush, restore RAT, new PC to fetch)
>   ```
>   The branch is resolved in an execution stage **~15 stages after it was fetched**, plus 1–2 cycles to flush and redirect fetch ⇒ **~16–18 cycles**. "17" is a representative value; measured minimum penalties on recent Intel/AMD cores are about 15–20 cycles (lower when the µop cache supplies the instructions, because the decode stages are skipped).
> - **Why so many stages?** At 4–5 GHz a cycle is only 0.2–0.25 ns, so each stage can do very little work. Fetching from the I-cache, decoding x86, renaming several instructions per cycle, and scheduling each need several stages.
> - **It can be much worse than the minimum.** The penalty counts from when the branch *could* execute. If the branch condition depends on a load that misses to DRAM, it sits in the scheduler for hundreds of cycles, and all the speculative work fetched meanwhile is thrown away on a mispredict.
> - **Lost work scales with width:** 17 cycles × 4 instructions/cycle ≈ **68 instruction slots** wasted per mispredict. That's why big cores invest so heavily in branch predictors ([05](05-Branch-Prediction.md)).

### 2.13 Exceptions and interrupts in a pipeline
**Exception** = unexpected event caused by an instruction (page fault, illegal instruction, divide by zero, misaligned access). **Interrupt** = external event (timer, I/O device).

**Precise exception** (required by all modern ISAs):
1. All instructions **before** the faulting one have **completed**.
2. **No** instruction after it has changed registers or memory.
3. The saved PC (`mepc`/`sepc` in RISC-V, EPC in MIPS, ELR on ARM) points to the faulting instruction.
4. A cause register (`mcause`/`scause`, ESR on ARM) records why.

Why precise exceptions matter: after a **page fault**, the OS loads the page and **restarts the same instruction**. That works only if the machine state is exactly "everything before it done, nothing after it done".

**How a 5-stage pipeline implements it:**
```
Cycle:          1   2   3   4   5
add             IF  ID  EX  MEM WB        ← older: allowed to complete
lw  (faults)        IF  ID  EX  MEM       ← page fault detected in MEM (cycle 5)
I3                      IF  ID  EX  flush ← younger: flushed
I4                          IF  ID  flush
I5                              IF  flush
                                          → save PC of lw, set cause, fetch from handler address
```
- Each stage that detects an exception **records it in the pipeline register** (with the instruction) instead of acting immediately.
- The exception is **acted on at one point** (the end of MEM / start of WB here; the ROB head in out-of-order CPUs).
- **Several exceptions in the same cycle?** Handle the **oldest** instruction's exception first. For example, the `lw` in MEM has a page fault and a younger instruction in ID has an illegal opcode: the page fault wins. The younger one is flushed and will raise its exception again when re-executed (if it is still reached).
- **Stores** must not write memory if an older instruction faults. In this pipeline, MEM is in order, so this is automatic.

### 2.14 Multi-cycle operations (floating point)
Some operations can't finish in one EX cycle. Instead of slowing the clock for everyone, they get their own longer functional units.

Classic example (H&P MIPS FP pipeline):
| Functional unit | Latency* | Initiation interval** |
|---|---|---|
| Integer ALU | 0 | 1 |
| Data memory (loads) | 1 | 1 |
| FP add | 3 | 1 |
| FP multiply | 6 | 1 |
| FP divide | 24 | 25 (not pipelined) |
\* **Latency** = number of cycles a dependent instruction must wait between producer and consumer.
\** **Initiation interval** = cycles between issuing two operations to the same unit (1 = fully pipelined).

New problems this creates:
1. **More RAW stalls:** an instruction using an FP multiply result waits up to 6 cycles.
2. **Structural hazard on the divider** (not pipelined): a second divide waits.
3. **Structural hazard on the register-file write port:** instructions of different latencies can reach WB in the same cycle.
4. **WAW hazards appear** because instructions finish **out of order**:
   ```
   fdiv.d f0, f2, f4     # finishes after ~24 cycles
   fadd.d f0, f6, f8     # finishes after ~3 cycles → writes f0 FIRST
   → then fdiv overwrites f0 → final f0 is the wrong (older) value ✗
   ```
   Fix: detect it in ID and stall the `fadd` (or cancel the `fdiv`'s write).
5. **Imprecise exceptions:** the `fadd` may have already written f0 when the older `fdiv` raises a divide-by-zero.
Historic solution: **scoreboarding** (CDC 6600). Modern solution: **reorder buffer** with in-order commit ([09](09-ILP-Superscalar-Out-of-Order.md)).

### 2.15 How deep should a pipeline be?
Splitting the work into more stages (**superpipelining**) shortens each stage. But:
| Deeper (more stages) | Shallower (fewer stages) |
|---|---|
| ✅ shorter stages ⇒ higher clock frequency | ❌ lower frequency |
| ❌ latch overhead becomes a bigger fraction of each cycle | ✅ less overhead |
| ❌ **bigger branch mispredict penalty** (resolved later) | ✅ smaller penalty |
| ❌ **bigger load-use and other data hazard penalties** | ✅ smaller |
| ❌ more power (more latches, higher frequency) | ✅ better performance per watt |

```
 Performance
      │        ___
      │      /     \___            ← optimum: latch overhead vs hazard penalties
      │    /            \___
      │  /                   \___
      └────────────────────────────── pipeline depth
        5     10    15    20    30
```
Real-world examples:
| Core | Pipeline | Notes |
|---|---|---|
| ARM Cortex-M0+ | 2 stages | ultra-low-power microcontroller |
| ARM Cortex-M3 / M4 | 3 stages (fetch, decode, execute) | TI Tiva/MSP432, STM32 |
| ARM Cortex-M7 | 6 stages, dual-issue | high-end MCU |
| ARM Cortex-A55 | ~8 stages, in-order | efficiency core in phones |
| Modern Intel / AMD / Apple big cores | ~14–20 stages, out-of-order | |
| Intel Pentium 4 (Prescott) | 31 stages | very high GHz, but hot and low IPC; the industry backed away from this |

Lesson from the Pentium 4: chasing GHz with very deep pipelines loses to better IPC once power and mispredict penalties are counted.

### 2.16 How to solve any pipeline stall question (recipe)
1. **List dependences:** for each instruction, which earlier instruction produces each source register? Note the distance.
2. **Check the assumptions:** forwarding or not? Where are branches resolved? Is the register file write-then-read in one cycle? Is there MEM→MEM forwarding for stores?
3. **Apply the stall tables:** 2.9 (no forwarding), 2.10 (forwarding), 2.11 (load-use), 2.12 (branches).
4. **Draw the diagram** cycle by cycle. A stall delays *every* younger instruction too, which can change later distances.
5. **Total cycles = k + (n − 1) + total stalls.** Double-check with the diagram's last WB.

**Default assumptions when the question doesn't say** (state them aloud before solving):
| Assumption | Default |
|---|---|
| Stages | IF, ID, EX, MEM, WB (or exactly the stages the question lists, e.g., IF-ID-OF-EX-WB) |
| Time per stage | 1 cycle; if stage delays are given, clock = slowest stage + latch delay |
| Ideal CPI | 1 |
| Memory | every access hits in 1 cycle (no cache misses unless mentioned) |
| Register file | write first half, read second half |
| Forwarding | as stated; if not stated, ask or solve both ways |
| Branches | resolved in EX, predict-not-taken ⇒ 2-cycle penalty when taken |
Say: *"I'll assume one cycle per stage, full forwarding, and branches resolved in EX with predict-not-taken. Is that okay?"*
And if asked about **real** CPUs: textbook penalty = 2 cycles; modern high-end cores ≈ 15–20 cycles (branch resolves ~15 stages after fetch).

### 2.17 In-order vs out-of-order is NOT about the number of stages
Three **independent** design choices:
| Dimension | Options | Question it answers |
|---|---|---|
| **Depth** | 2 … 30+ stages | How many steps is the work split into? |
| **Width** | scalar (1/cycle) vs superscalar (2–10/cycle) | How many instructions per cycle? |
| **Order** | **in-order** vs **out-of-order** | Can a later instruction *execute* before an earlier stalled one? |

- **In-order:** instructions execute in program order. If one stalls (e.g., waiting for a load), **everything behind it waits**, even independent instructions.
- **Out-of-order (OoO):** independent later instructions **execute past** the stalled one. Results are still **committed in program order** (via the ROB), so the program can't tell.

```
lw  x1, 0(x2)     # cache miss: waits many cycles
add x3, x1, x4    # depends on x1: must wait in both designs
sub x5, x6, x7    # independent
In-order:     sub waits behind add                → wasted cycles
Out-of-order: sub executes while lw is waiting    → useful work during the miss
```

**Real examples (any depth can be either):**
| | Shallow | Deep |
|---|---|---|
| **In-order** | classic 5-stage; ARM Cortex-M0+ (2 stages), Cortex-M3/M4 (3); original Pentium (5 stages, 2-wide superscalar) | Cortex-A53/A55 (~8, dual-issue), Intel Atom "Bonnell" (16), Itanium 2 (8), GPU SMs |
| **Out-of-order** | ARM Cortex-A9 (~8–11) | Intel Golden Cove, AMD Zen 4, Apple Firestorm (~14–20) |

**What does change in an OoO pipeline is the *kind* of stages:** extra stages are needed to make OoO work, so OoO pipelines also tend to be deeper.
```
5-stage in-order:  IF → ID → EX → MEM → WB
Out-of-order:      Fetch → Decode → RENAME → DISPATCH → ISSUE (wait in scheduler) → Reg read → EX/MEM → Writeback → COMMIT
                   └──────── in order ────────┘   └──────────── out of order ────────────┘           └ in order ┘
```
| 5-stage | OoO equivalent | New in OoO |
|---|---|---|
| IF | Fetch (several stages, with branch prediction) | |
| ID | Decode | **Rename** (map to physical registers, removes WAR/WAW), **Dispatch** (allocate ROB / scheduler / load-store queue entries) |
| — | | **Issue / schedule**: wait until operands are ready, then go (this is where the reordering happens) |
| EX, MEM | Execute units, load/store unit | **Load/store queue** checks memory ordering |
| WB | Writeback to the physical register file | **Commit / retire**: in program order from the ROB head (precise exceptions) |
Summary: an OoO core is **in-order at the front (fetch → dispatch), out-of-order in the middle (issue → execute), and in-order at the back (commit)**. Details in [09](09-ILP-Superscalar-Out-of-Order.md).

**Subtle point:** even an in-order pipeline can **complete** out of order when it has multi-cycle units (a 3-cycle FP add finishes before an earlier 24-cycle divide; see 2.14). That's "in-order issue, out-of-order completion". It's not an OoO core, because instructions still *start* executing in program order.

---

## 3. Worked Numericals

**N1. Single-cycle vs multi-cycle vs pipelined.** Step delays from 2.1. Mix: 25% `lw`, 10% `sw`, 52% R-type, 13% `beq`.
- **Single-cycle:** clock = 800 ps, CPI = 1 ⇒ **800 ps/instruction**.
- **Multi-cycle:** clock = 200 ps; cycles: `lw` 5, `sw` 4, R-type 4, `beq` 3.
  CPI = 0.25×5 + 0.10×4 + 0.52×4 + 0.13×3 = 1.25 + 0.40 + 2.08 + 0.39 = **4.12** ⇒ 4.12 × 200 = **824 ps/instruction**. Slightly *worse* than single-cycle!
- **Pipelined:** clock = 200 ps, CPI ≈ 1 ⇒ **200 ps/instruction** ⇒ **4×** faster than single-cycle. (Not 5×, because the 100 ps register stages are padded to 200 ps.)

**N2. Pipeline speedup with latch overhead.** Stage delays IF 200, ID 150, EX 250, MEM 200, WB 100 ps; latch 20 ps; 1000 instructions.
- Non-pipelined: 200 + 150 + 250 + 200 + 100 = 900 ps/instruction ⇒ 1000 × 900 = **900 ns**.
- τ = 250 + 20 = 270 ps. Pipelined: (5 + 1000 − 1) × 270 = 1004 × 270 = 271,080 ps ≈ **271 ns**.
- Speedup = 900 / 271.08 ≈ **3.32**. As n → ∞: 900 / 270 = 3.33. It is not 5, because of imbalance (EX = 250 vs WB = 100) and latch overhead.
- Ideal throughput = 1 / 270 ps ≈ **3.7 billion instructions/s**.

**N3. Should we split the slowest stage?** Same as N2, but EX is split into two stages of 125 ps (6 stages total).
- New τ = max(200, 150, 125, 125, 200, 100) + 20 = **220 ps**. Asymptotic speedup = 900 / 220 = **4.09** (vs 3.33).
- Catch: if the ALU result is now ready only after EX2, a dependent instruction right after an ALU instruction needs **1 stall** even with forwarding (like a load-use today), and branches resolve a cycle later. If that adds 0.2 CPI: time/instr = 1.2 × 220 = 264 ps vs 270 ps. Only a **2%** gain. Deeper isn't automatically better.

**N4. Without vs with forwarding.**
```
I1: add x1, x2, x3
I2: sub x4, x1, x5      (needs x1 from I1, distance 1)
I3: and x6, x4, x7      (needs x4 from I2, distance 1)
I4: or  x8, x1, x6      (needs x6 from I3, distance 1; x1 from I1, distance 3)
```
**Without forwarding** (2 stalls per distance-1 dependence):
```
Cycle:          1   2   3   4   5   6   7   8   9   10  11  12  13  14
add x1,x2,x3    IF  ID  EX  MEM WB
sub x4,x1,x5        IF  ID  ID* ID* EX  MEM WB
and x6,x4,x7            IF  IF* IF* ID  ID* ID* EX  MEM WB
or  x8,x1,x6                        IF  IF* IF* ID  ID* ID* EX  MEM WB
```
6 stalls ⇒ 5 + 3 + 6 = **14 cycles**.
**With forwarding:**
```
Cycle:          1   2   3   4   5   6   7   8
add x1,x2,x3    IF  ID  EX  MEM WB
sub x4,x1,x5        IF  ID  EX  MEM WB          ← x1 via EX/MEM → EX
and x6,x4,x7            IF  ID  EX  MEM WB      ← x4 via EX/MEM → EX
or  x8,x1,x6                IF  ID  EX  MEM WB  ← x6 via EX/MEM → EX; x1 from register file (written cycle 5)
```
0 stalls ⇒ **8 cycles** ⇒ forwarding gives a **1.75×** speedup on this sequence.

**N5. CPI with hazards.** 5-stage with forwarding. 30% loads, and half of them are immediately followed by an instruction that uses the loaded value (1 stall each). 20% branches, predict-not-taken, 60% of branches taken, penalty 2 cycles.
- Load-use stalls per instruction = 0.30 × 0.5 × 1 = 0.15
- Branch stalls per instruction = 0.20 × 0.6 × 2 = 0.24
- CPI = 1 + 0.15 + 0.24 = **1.39**. Real throughput at τ = 270 ps: 1 / (1.39 × 270 ps) ≈ **2.66 billion instructions/s** (vs 3.7 ideal).

**N6. Resolve branches in EX or in ID?** Same mix as N5 (20% branches, 60% taken). Moving the comparison to ID cuts the taken penalty to 1 cycle, but 30% of branches use a register computed by the immediately preceding ALU instruction (1 extra stall), and the ID stage gets 10% longer, stretching the clock by 10%.
- In EX: branch CPI = 0.2 × 0.6 × 2 = 0.24 ⇒ CPI = 1.24 ⇒ time = **1.24τ**
- In ID: 0.2 × 0.6 × 1 + 0.2 × 0.3 × 1 = 0.12 + 0.06 = 0.18 ⇒ CPI = 1.18; clock 1.1τ ⇒ time = 1.18 × 1.1τ = **1.298τ**
⇒ Moving the branch to ID is **4.7% slower** here, despite the smaller penalty. Always check the effect on the clock period.

**N7. Deep vs shallow pipeline.** 20% branches.
- No prediction (every branch pays the full penalty):
  - A: 5 stages, 1 GHz, penalty 2 ⇒ CPI = 1 + 0.2 × 2 = 1.4 ⇒ **1.40 ns/instruction**
  - B: 15 stages, 2.5 GHz, penalty 10 ⇒ CPI = 1 + 0.2 × 10 = 3.0 ⇒ 3.0 / 2.5 = **1.20 ns/instruction**
- With a predictor (5% misprediction rate):
  - A: CPI = 1 + 0.2 × 0.05 × 2 = 1.02 ⇒ **1.02 ns**
  - B: CPI = 1 + 0.2 × 0.05 × 10 = 1.1 ⇒ 1.1 / 2.5 = **0.44 ns**
⇒ The deep pipeline wins **2.3×** with a good predictor but only 1.17× without. *Deep pipelines are only worthwhile with good branch prediction.*

**N8. Identify all dependences and count stalls.**
```
I1: lw   x1, 0(x2)
I2: add  x3, x1, x1      RAW x1 (I1→I2), distance 1, producer is a load → 1 stall
I3: sub  x1, x4, x5      WAW x1 (I1, I3); WAR x1 (I2 reads, I3 writes) → harmless in-order
I4: sw   x3, 4(x1)       RAW x3 (I2→I4), distance 2 → forward; RAW x1 (I3→I4), distance 1 → forward
```
Stalls with forwarding: **1** (I1→I2 only) ⇒ 5 + 3 + 1 = **9 cycles**.

**N9. Loop timing.** 5-stage, forwarding, branch resolved in EX, predict-not-taken (penalty 2 when taken).
```asm
loop: lw   x5, 0(x10)      # load a[i]
      add  x6, x6, x5      # sum += a[i]   (uses x5 right after the load → 1 stall)
      addi x10, x10, 8     # next element
      bne  x10, x11, loop  # taken 99 times, not taken once
```
- Each iteration: 4 instructions + 1 load-use stall + 2 cycles taken-branch penalty = **7 cycles**.
- Last iteration: branch not taken ⇒ 4 + 1 = 5 cycles.
- Total = 99 × 7 + 5 + 4 (pipeline fill) = 693 + 9 = **702 cycles** for 400 instructions ⇒ CPI ≈ 1.75.
- **Optimise:** move `addi` between `lw` and `add` (the `lw` uses x10 before `addi` changes it, so this is safe) ⇒ no load-use stall ⇒ 99 × 6 + 4 + 4 = **602 cycles** (14% faster, zero hardware change).
- Scheduled order **plus** a branch predictor (with a BTB) that's right on every iteration except the last: 400 instructions + 4 (fill) + 2 (one mispredict at loop exit) = **406 cycles** ⇒ CPI ≈ 1.0.

**N10. Multi-cycle stall count.** Using the latencies in 2.14 (FP multiply latency 6, FP add latency 3), forwarding available:
```
fmul.d f0, f2, f4
fadd.d f6, f0, f8     # needs f0: waits 6 cycles
fsd    f6, 0(x1)      # needs f6 as store data: FP add → store latency is 2 in H&P's table (store needs data one stage later)
```
Stalls = 6 + 2 = **8**. Rule: you pay the latency of each producer in the chain, unless independent instructions are scheduled in between.

---

## 4. Interview Questions

### Basic
- **Q: Does pipelining reduce the latency of an instruction?** No. Each instruction still passes through all stages, plus latch overhead, so latency rises slightly. Pipelining improves **throughput**.
- **Q: What are the 5 stages and what happens in each?** See 2.2. Mention which hardware each uses.
- **Q: Name the three hazard types with an example of each.** Structural (shared memory for IF/MEM), data (`add x1` then `sub` using x1), control (`beq`).
- **Q: What is forwarding?** Sending a result from the EX/MEM or MEM/WB pipeline register straight to the ALU input of a dependent instruction, so it doesn't wait for write-back.
- **Q: Why do we need pipeline registers?** To hold each instruction's data and control signals between stages, so 5 instructions can be in flight without interfering.

### Intermediate
- **Q: Why can't forwarding eliminate the load-use stall?** The load's data appears at the end of MEM, but the consumer needs it at the start of EX in that same cycle. One bubble is unavoidable in hardware; the compiler can hide it by scheduling.
- **Q: Write the forwarding condition.** See 2.10. Explain `rd != 0` and why EX/MEM has priority over MEM/WB.
- **Q: What happens in hardware during a stall?** PC and IF/ID are frozen, and ID/EX control bits are zeroed to insert a bubble (2.11).
- **Q: Why don't WAR/WAW hazards occur in the classic 5-stage pipeline?** Reads happen early (ID), writes happen late (WB), and everything moves in order. They appear with multi-cycle units or out-of-order execution.
- **Q: What is a precise exception? How does a pipeline implement it?** See 2.13: record exceptions in pipeline registers, handle them at one point in program order, flush younger instructions, save the PC and the cause.
- **Q [TI]: Speedup of a k-stage pipeline for n instructions?** nk / (k + n − 1), which approaches k as n → ∞ (balanced stages, no hazards).
- **Q [TI]: What is a branch delay slot? Why do DSPs keep them?** The instruction(s) after a branch always execute. DSPs (e.g., TI C6000 with 5 branch delay slots) are programmed by compilers that can fill the slots, and it avoids branch-prediction hardware (simpler, lower power, deterministic timing).

### Advanced / killer follow-ups
- **Q [APPLE][NVIDIA]: Why not make the pipeline 50 stages deep?** Latch overhead dominates short stages, mispredict and load-use penalties explode, power rises (more latches, higher frequency), and wire delay and clock skew limit the gain (2.15, N7).
- **Q: Design the hazard-detection unit for load-use.** Condition: `ID/EX.MemRead && (ID/EX.rd == IF/ID.rs1 || ID/EX.rd == IF/ID.rs2)`. Actions: freeze PC and IF/ID, zero the ID/EX control bits.
- **Q: Would you move branch resolution to ID?** It halves the taken penalty, but needs forwarding into ID, adds stalls when the branch depends on the previous ALU result or a load, and may stretch the clock (N6).
- **Q [HFT]: Why is a branch misprediction on a modern x86 ~15–20 cycles?** The branch resolves deep in the pipeline (after fetch, decode, rename, scheduling, execute). On a 4–6-wide core that wastes 60–100+ instruction slots.
- **Q: Why did delayed branches fall out of favour?** They expose a microarchitecture detail (the 1-cycle penalty of a 5-stage pipeline) in the ISA forever; in deeper or superscalar pipelines one slot doesn't help.
- **Q: Two exceptions occur in the same cycle in different stages. Which is handled first?** The one from the **older** instruction (further along the pipeline), so exceptions are handled in program order.
- **Q: How do multi-cycle FP units break the simple pipeline?** Out-of-order completion ⇒ WAW hazards, write-port conflicts, and imprecise exceptions (2.14). Solutions: scoreboard or ROB.
- **Q [NVIDIA]: How do GPUs deal with pipeline hazards without forwarding everything?** They switch to another warp every cycle (fine-grained multithreading): while one warp waits for its dependency, others issue. See [12](12-GPU-Architecture.md).

---

## 5. Common Traps
- Using k (number of stages) as the speedup without considering imbalance and latch overhead.
- Forgetting pipeline fill: total cycles = k + n − 1 (+ stalls), not n.
- Claiming forwarding fixes *all* data hazards (load-use still needs 1 stall).
- Forgetting that a stall delays **all** younger instructions. Redraw the diagram, don't just add numbers blindly.
- Double-counting: ideal CPI = 1, and stalls are **added** to it.
- Forgetting the `rd != 0` check, or giving MEM/WB priority over EX/MEM in forwarding.
- Confusing the names: WAR = Write After Read = anti-dependence; RAW = true dependence.
- Assuming the branch penalty applies to all branches with predict-not-taken. It applies only to **taken** branches.
- Moving work to an earlier stage (e.g., branches into ID) without checking the clock period (N6).

---

## 6. Self-Check Quiz
1. In a 5-stage pipeline **without forwarding**, how many stalls does an instruction need if it uses a result produced by the instruction **two** before it?
2. `lw x1, 0(x2)` is followed immediately by `sw x1, 0(x3)`. With full forwarding including MEM→MEM, how many stalls?
3. A pipeline has stages of 300, 200, 200, 250, 150 ps and 25 ps latch overhead. What is the clock period, and the maximum speedup over the unpipelined version?
4. Branches are 25% of instructions, 70% are taken, predict-not-taken, penalty 3 cycles. What is the branch CPI contribution?
5. Why is the `rd != 0` check needed in the forwarding unit?

<details><summary>Answers</summary>

1. **1 stall** (distance 2 without forwarding; see the 2.9 table).
2. **0 stalls.** The store needs the data only in MEM, and MEM/WB → MEM forwarding delivers it in time.
3. τ = 300 + 25 = **325 ps**. Unpipelined = 1100 ps ⇒ max speedup = 1100 / 325 ≈ **3.38**.
4. 0.25 × 0.7 × 3 = **0.525** cycles per instruction.
5. x0 is hard-wired to zero. An instruction that "writes" x0 (e.g., `addi x0, x0, 5`, often used as a NOP) must not forward its non-zero result to a consumer reading x0.
</details>

---

## 7. Revision Summary
| | |
|---|---|
| Why pipeline | short clock (multi-cycle) + CPI ≈ 1 (single-cycle) |
| Stages | IF (fetch) · ID (decode, read regs) · EX (ALU) · MEM (D-cache) · WB (write reg) |
| Pipeline registers | carry data **and control** with each instruction; NOP = all control bits 0 |
| τ | max stage delay + latch overhead |
| Time | (k + n − 1)τ; speedup → T_total / τ ≤ k |
| With stalls | CPI = 1 + structural + data + control stalls |
| Hazards | structural (shared hardware) · data (RAW true; WAR/WAW name only) · control (next PC unknown) |
| No forwarding | distance 1 → 2 stalls, 2 → 1, 3 → 0 |
| Forwarding | EX/MEM → EX (1 ahead), MEM/WB → EX (2 ahead); check RegWrite, rd ≠ 0, newest wins |
| Load-use | 1 stall even with forwarding; stall = freeze PC + IF/ID, bubble into ID/EX; fix by scheduling |
| Branch in EX | 2-cycle penalty (predict-not-taken: only when taken); in ID: 1 cycle but more data stalls |
| Branch CPI | branch freq × mispredict rate × penalty (~15–20 cycles on modern CPUs) |
| Precise exceptions | handle at one point, oldest first, flush younger, save PC + cause |
| Multi-cycle units | out-of-order completion ⇒ WAW, write-port conflicts, imprecise exceptions ⇒ scoreboard / ROB |
| Depth | deeper ⇒ ↑f but ↑penalties, ↑power; P4's 31 stages was too deep; today ~14–20 |
