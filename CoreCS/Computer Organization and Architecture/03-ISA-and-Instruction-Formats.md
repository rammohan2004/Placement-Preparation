# 03. Instruction Set Architecture (ISA) & Instruction Formats

> **Asked at:** [QCOM] [APPLE] ARM vs x86, RISC vs CISC · [TI] endianness, alignment, Thumb, addressing modes · [DESHAW] [HFT] calling conventions, struct padding · GATE-style encoding numericals in OAs.
> **Prerequisites:** [01](01-Number-Representation-and-Arithmetic.md).

---

## 1. Intuition
The ISA is the **contract** between software and hardware: the instructions, registers, memory model and encoding a compiler may rely on. The **microarchitecture** is *how* one chip fulfils the contract. Intel and AMD build very different machines that run the same x86-64 binaries; Apple M-series and Qualcomm Oryon both implement ARMv8/9-A.

> ISA = *what* (visible to programmer) · Microarchitecture = *how* (pipeline, caches, predictors — invisible except for speed).

### ISA vs Microarchitecture vs Processor
These are **three layers**, from most abstract to most physical:

```
 ISA (specification)            "what instructions exist and what they mean"
   │  implemented by
   ▼
 Microarchitecture (design)     "how the hardware executes those instructions"
   │  manufactured as
   ▼
 Processor (physical chip)      "the actual silicon product you buy"
```

**Analogy: cars**
- **ISA** = the *driving interface*: steering wheel, accelerator, brake, gear positions. Any licensed driver can drive any car that follows it. (Compiled programs are the drivers.)
- **Microarchitecture** = the *engine and drivetrain design*: petrol or electric, number of cylinders, turbo. The driver never sees it but feels the performance.
- **Processor** = the *specific car model on the road*: that engine design built at a specific factory, with a given tuning, fuel tank, and price.

| | **ISA** | **Microarchitecture** | **Processor** |
|---|---|---|---|
| What it is | Abstract **specification** (a document / contract) | A **hardware design** that implements an ISA | A **physical chip** (product) built from one or more microarchitectures |
| Defines | Instructions, registers, data types, addressing modes, encoding, memory model, privilege levels, exceptions | Pipeline depth, issue width, OoO or in-order, branch predictor, cache organisation, ROB size, execution units | Process node (e.g., TSMC 3 nm), clock frequency, core count, cache **sizes**, memory controllers, GPU/NPU on die, power limits, packaging |
| Visible to software? | **Yes**: the compiler/assembler targets it | **No**, only through performance (and side channels) | Partly: core count, frequency, features reported by `cpuid` / `/proc/cpuinfo` |
| Changes affect | **Binary compatibility** (new ISA = recompile) | Speed, power, area; same binaries still run | Price, performance tier, power envelope (SKU / binning) |
| Who creates it | Arm Ltd., Intel/AMD (x86), RISC-V International | CPU design teams (Apple, Arm, Intel, AMD, Qualcomm, SiFive) | Chip companies + foundries |
| Lifespan | Decades (x86 since 1978, ARM since 1985) | A few years per generation | One product generation; many SKUs |
| Who uses it | Compiler writers, OS developers, assembly programmers | CPU architects, performance engineers | System builders, buyers |

**Real examples: one ISA → many microarchitectures → many processors**
| ISA | Microarchitecture (core design) | Processor (chip you buy) |
|---|---|---|
| x86-64 | AMD **Zen 4** | AMD Ryzen 9 7950X, EPYC 9654 |
| x86-64 | Intel **Golden Cove** (P-core) + **Gracemont** (E-core) | Intel Core i9-12900K (Alder Lake) |
| ARMv8-A | Apple **Firestorm** (P) + **Icestorm** (E) | Apple **M1**, A14 Bionic |
| ARMv8.2-A | Arm **Cortex-X1 / A78 / A55** (licensed designs) | Qualcomm **Snapdragon 888** |
| ARMv7-M / v8-M (Thumb) | Arm **Cortex-M4 / M33** | TI **Tiva C / MSP432**, STM32 MCUs |
| RISC-V (RV64GC) | SiFive **U74** | SiFive / StarFive JH7110 |
| (GPU) PTX → SASS | NVIDIA **Hopper** | NVIDIA **H100** |

Key takeaways for interviews:
1. **Same ISA, different microarchitecture** ⇒ the same binary runs on both, at different speed and power. (Cortex-A55 is in-order and tiny; Apple Firestorm is 8-wide OoO. Both run the same ARM64 code.)
2. **Same microarchitecture, different processors** ⇒ the same core design is shipped in different chips with different core counts, clocks, cache sizes and prices (Zen 4 in both a desktop Ryzen and a 96-core EPYC).
3. **ARM's business model shows the split clearly.** Arm Ltd. licenses the **ISA** (architecture licence: Apple and Qualcomm design their *own* microarchitectures) or a ready-made **microarchitecture** (Cortex cores). Companies then build their own **processors** (SoCs) around them.
4. "Architecture" is often used loosely to mean the ISA ("ARM architecture") *or* the microarchitecture ("Zen architecture"). In an interview, say which one you mean.

---

## 2. Deep Dive

### 2.1 What an ISA specifies
- Instruction set & semantics (incl. exceptions)
- Registers (count, width, special ones: PC, SP, flags)
- Data types & sizes
- Addressing modes
- Instruction encoding
- **Memory model** (ordering rules — see [10](10-Cache-Coherence-and-Memory-Consistency.md)), endianness, alignment
- Privilege levels, interrupts/exceptions, virtual memory interface

### 2.2 ISA classes (by where operands live)
| Class | Example `C = A + B` | Example machines |
|---|---|---|
| Stack | `push A; push B; add; pop C` | JVM bytecode, x87 FPU |
| Accumulator | `load A; add B; store C` | early micros, 8051-ish |
| Register-memory | `mov r1, A; add r1, B; mov C, r1` | x86 |
| **Load-store (reg-reg)** | `lw r1,A; lw r2,B; add r3,r1,r2; sw r3,C` | RISC-V, ARM, MIPS |

### 2.3 RISC vs CISC
| | RISC (ARM, RISC-V, MIPS) | CISC (x86) |
|---|---|---|
| Instruction length | fixed (32-bit; 16-bit compressed variants) | variable, 1–15 bytes |
| Memory access | only load/store | most instructions can touch memory |
| Addressing modes | few | many, complex |
| Registers | 31–32 GPRs | 16 GPRs (x86-64) |
| Decode | simple, easy to decode many in parallel | complex; uses µop cache |
| Code density | lower (fixed by Thumb/RVC) | higher |

**The modern reality (great interview point):** x86 cores decode CISC instructions into RISC-like **micro-ops (µops)** and run them on a RISC-style out-of-order core. The ISA difference is now mostly about **decode cost and power**, not performance ceilings. Fixed-length ARM lets Apple's cores decode **8–10 instructions/cycle**; x86 relies on a µop cache to avoid re-decoding.

### 2.4 Instruction formats — RISC-V RV32I (32-bit fixed)
```
 31        25 24  20 19  15 14 12 11        7 6      0
 | funct7    | rs2  | rs1  |funct3| rd        | opcode |  R-type  add  rd, rs1, rs2
 | imm[11:0]        | rs1  |funct3| rd        | opcode |  I-type  addi, lw, jalr
 | imm[11:5] | rs2  | rs1  |funct3| imm[4:0]  | opcode |  S-type  sw
 | imm[12|10:5]|rs2 | rs1  |funct3|imm[4:1|11]| opcode |  B-type  beq (offset ×2)
 | imm[31:12]                     | rd        | opcode |  U-type  lui, auipc
 | imm[20|10:1|11|19:12]          | rd        | opcode |  J-type  jal
```
Design choices worth mentioning:
- **rs1, rs2, rd always in the same bit positions** → register file can be read *before* decode finishes.
- Sign bit of immediates always bit 31 → sign extension starts early.
- No condition flags: `beq/blt` compare registers directly (flags create hidden dependencies that complicate OoO).
- 5-bit register fields ⇒ 32 registers; x0 hard-wired to 0.

### 2.5 Addressing modes
| Mode | Example | Meaning | Use |
|---|---|---|---|
| Immediate | `addi x1, x2, 5` | operand in instruction | constants |
| Register | `add x1, x2, x3` | operand in register | |
| Direct (absolute) | `lw x1, 0x1000` | M[0x1000] | globals (CISC) |
| Register indirect | `lw x1, 0(x2)` | M[x2] | pointers |
| Base + displacement | `lw x1, 8(x2)` | M[x2 + 8] | struct fields, stack |
| Indexed | `mov eax, [rbx+rcx]` | M[rb + ri] | arrays |
| Scaled index | `mov eax, [rbx+rcx*4+8]` | M[rb + 4·ri + 8] | `a[i]` of ints (x86) |
| PC-relative | `beq x1,x2, L` / `auipc` | PC + offset | branches, position-independent code |
| Auto-inc/dec | `ldr r0, [r1], #4` (ARM post-index) | use then r1 += 4 | stack, array walk, DSP loops |
| Memory indirect | `M[M[x]]` | double lookup | rare (old CISC) |

DSPs (TI C6000, Qualcomm Hexagon) add **circular/modulo addressing** and **bit-reversed addressing** (for FFT) in hardware.

### 2.6 Endianness [TI][QCOM][DESHAW]
Store 32-bit `0x12345678` at address 0x100:
| Address | 0x100 | 0x101 | 0x102 | 0x103 |
|---|---|---|---|---|
| **Little-endian** (x86, ARM default, RISC-V) | 78 | 56 | 34 | 12 |
| **Big-endian** (network order, old PowerPC, some DSP modes) | 12 | 34 | 56 | 78 |

```c
int is_little_endian(void) {
    uint32_t x = 1;
    return *(uint8_t *)&x == 1;   // LSB stored at lowest address
}
```
- Network protocols are big-endian ⇒ `htonl/ntohl` (HFT market-data feeds often specify byte order explicitly!).
- Endianness matters only when you look at a multi-byte value **byte-by-byte** (casting, serialisation, DMA buffers, memory dumps).
- Swap: `__builtin_bswap32(x)` (single `BSWAP`/`REV` instruction).

### 2.7 Alignment and struct padding [DESHAW][HFT][TI]
An access of size k is aligned if address % k == 0. Misaligned access: slower (may split across cache lines), or a **fault** on some architectures (older ARM, many DSPs).
```c
struct A { char a; int b; char c; };   // 1 + 3 pad + 4 + 1 + 3 pad = 12 bytes
struct B { int b; char a; char c; };   // 4 + 1 + 1 + 2 pad          =  8 bytes
```
Rule: each member aligned to its own size; struct size rounded up to largest member alignment. **Order members largest → smallest** to minimise padding. HFT code also aligns hot structs to 64 B (`alignas(64)`) to avoid false sharing (see [10](10-Cache-Coherence-and-Memory-Consistency.md)).

### 2.8 Procedure calls & calling conventions
| | RISC-V | x86-64 System V (Linux) | AArch64 |
|---|---|---|---|
| Args | a0–a7 | rdi, rsi, rdx, rcx, r8, r9 | x0–x7 |
| Return | a0, a1 | rax (rdx) | x0 |
| Return address | ra (x1) register | pushed on stack by `call` | x30 (LR) |
| Callee-saved | s0–s11, sp | rbx, rbp, r12–r15 | x19–x28 |
| Caller-saved | t0–t6, a0–a7 | rax, rcx, rdx, rsi, rdi, r8–r11 | x0–x18 |

- **Caller-saved** (scratch): callee may clobber; caller saves if it needs them after the call.
- **Callee-saved**: callee must restore before returning.
- A **leaf function** (calls nothing) can avoid saving the return address — cheap.
- Stack frame: return address, saved registers, locals, outgoing args; stack grows downward; x86-64 SysV keeps a 128-byte **red zone** below rsp for leaf functions.

#### 2.8.1 Caller-saved vs callee-saved registers (detailed explanation)

**Step 1: the problem.**
The CPU has only **one set of registers** (32 on RISC-V), and *every* function uses that same set. Suppose function `f` keeps an important value in register `x9` and then calls `g`. If `g` also uses `x9` for its own work, `f`'s value is overwritten. When `g` returns, `f` reads garbage.

```
 f:  x9 = 100          ← f's important value
     call g ─────────►  g:  x9 = 7      ← g overwrites x9 for its own use
                            ret
     use x9  ◄─────────     (f expects 100, gets 7) ❌
```
The fix is to **save the register to the stack (memory) and restore it later**. The question is **who** does the saving: the function making the call (**caller**) or the function being called (**callee**)?
The **calling convention** (part of the ABI) answers this by splitting registers into two groups, so that functions compiled separately (by different people, even different compilers) work together.

**Step 2: the two rules.**

| | **Caller-saved** registers | **Callee-saved** registers |
|---|---|---|
| Other names | volatile, scratch, temporary, "call-clobbered" | non-volatile, preserved, saved, "call-preserved" |
| Promise | **No promise.** The callee may freely overwrite them. | The callee **promises** they will hold the same value after it returns. |
| Who saves | The **caller**, *only if* it still needs the value after the call | The **callee**, *only if* it wants to use that register |
| When | Save just **before the call**, restore just **after** | Save in the callee's **prologue** (start), restore in its **epilogue** (end) |
| Best used for | Short-lived temporaries **not** needed after a call | Values that must **survive across calls** (loop counters, pointers, running sums) |
| RISC-V | `t0–t6`, `a0–a7`, `ra` | `s0–s11`, `sp` |
| x86-64 SysV | `rax, rcx, rdx, rsi, rdi, r8–r11` | `rbx, rbp, r12–r15`, `rsp` |
| AArch64 | `x0–x18` (+ `x30`/LR) | `x19–x28`, `x29`/FP, `sp` |

> **Read the names carefully:** "caller-saved" = *if anyone* saves it, it's the caller. "Callee-saved" = *if anyone* saves it, it's the callee. **Neither one means "always saved"**: a register is saved only when it is actually needed (caller-saved) or actually used (callee-saved).

**Analogy: lending your kitchen to a guest cook.**
- **Counter top = caller-saved.** The guest may use the counter and leave it messy. If *you* left something valuable on it, *you* must put it in the fridge before the guest arrives, and take it out afterwards.
- **Cupboards = callee-saved.** The guest may use the cupboards, but must put everything back **exactly as it was** before leaving. You don't need to do anything.

**Step 3: a worked example (RISC-V).**
```c
long g(long x) { return x * 2; }        // leaf function: calls nothing

long f(long a, long b) {
    long t = a + b;     // t is needed AFTER the call to g  → it must survive the call
    long u = g(a);
    return t + u;
}
```
**Option A: keep `t` in a callee-saved register (`s0`).** This is what compilers usually do.
```asm
f:  addi sp, sp, -16
    sd   ra, 8(sp)      # save return address: our own "call g" will overwrite ra
    sd   s0, 0(sp)      # we want to use s0, and s0 is callee-saved → f must preserve it for ITS caller
    add  s0, a0, a1     # t = a + b, kept in s0
    call g              # a0 = a is already the argument. g may destroy t0–t6, a0–a7, ra, but NOT s0
    add  a0, s0, a0     # return t + u  (s0 still holds t ✅)
    ld   s0, 0(sp)      # restore s0 for f's caller
    ld   ra, 8(sp)
    addi sp, sp, 16
    ret

g:  slli a0, a0, 1      # uses only a0 (caller-saved) → no saves needed at all
    ret
```
**Option B: keep `t` in a caller-saved register (`t0`).**
```asm
    add  t0, a0, a1     # t = a + b in t0
    sd   t0, 0(sp)      # caller (f) must save t0 BEFORE the call: g is allowed to destroy it
    call g
    ld   t0, 0(sp)      # and restore it AFTER the call
    add  a0, t0, a0
```
Both are correct. The difference is cost, and it shows up when there are **many calls**:
- Option B pays a save and a restore **around every call**.
- Option A pays once in the prologue and once in the epilogue, **no matter how many calls** happen in between.

**Step 4: why have both groups? Why not a single rule?**
| If **all** registers were caller-saved | If **all** registers were callee-saved |
|---|---|
| A function with a loop that calls `g` 1000 times must save/restore every live value **1000 times** | Every small **leaf** function (which calls nothing) must save every register it touches, even when its caller didn't care about their values |
| Bad for **non-leaf** functions | Bad for **leaf** functions, which are the most common kind |

With the split, the compiler picks the cheapest place for each value:
- Value **not needed after any call** (a temporary) → **caller-saved** register → usually **zero** saves (nothing to preserve).
- Value **needed after a call** → **callee-saved** register → saved **once** per function, not once per call.
- Leaf functions use only caller-saved registers → **no saves at all**. That is why leaf functions are cheap.

**Step 5: special registers.**
- **Return address (`ra` / LR):** effectively caller-saved. A non-leaf function **must** save it, because its own `call` overwrites it (see `sd ra, 8(sp)` above). Leaf functions don't touch it.
- **Stack pointer (`sp`):** callee-saved. It must hold the same value on return (the callee pops everything it pushed).
- **Argument and return registers (`a0–a7`, `rdi…`, `rax`):** caller-saved. The callee obviously writes the return value into them.

**Step 6: stack frame during the call** (stack grows downward):
```
 higher addresses
 ┌──────────────────────────┐
 │ caller's frame           │
 │  … caller-saved values   │ ← caller spilled these before "call" (if needed after)
 ├──────────────────────────┤ ← sp at the moment of the call
 │ return address (ra)      │ ← saved by callee if non-leaf (x86: pushed by `call` itself)
 │ saved s0, s1 …           │ ← callee-saved registers the callee will use (prologue)
 │ callee's local variables │
 └──────────────────────────┘ ← sp inside callee
 lower addresses
```

**Step 7: where this shows up in interviews and real systems.**
- **Interrupt handlers [TI][QCOM]:** an interrupt can arrive *between any two instructions*. The interrupted code made no call, so it saved nothing. The ISR must therefore preserve **all** registers, including caller-saved ones. On **ARM Cortex-M**, the hardware automatically pushes R0–R3, R12, LR, PC and xPSR (exactly the AAPCS caller-saved set) on interrupt entry. So the ISR can be an **ordinary C function**: the compiler already saves R4–R11 (callee-saved) if the ISR uses them. A classic embedded interview question.
- **Context switch (OS):** saves **all** registers of the outgoing thread. (In a cooperative switch implemented as a function call, only callee-saved registers + sp + ra need saving, because the caller-saved ones were already handled by the call convention.)
- **`setjmp`/`longjmp`:** `setjmp` stores only the **callee-saved** registers, sp and the return address. Caller-saved values are not guaranteed after `longjmp`, which is why locals modified after `setjmp` must be `volatile`.
- **Performance [HFT]:** each call costs saves/restores + the call/return. **Inlining** removes the call boundary, so the compiler no longer has to respect the convention for that call. Small hot functions should be inlinable (in a header, `inline`, LTO).
- **Mixing assembly and C:** hand-written assembly that forgets to restore a callee-saved register corrupts the caller in subtle, hard-to-debug ways.

**Quick self-check** (answers below)
1. `f` keeps a loop counter in `t1` (RISC-V) and calls `printf` inside the loop. What goes wrong, and how would you fix it?
2. Leaf function `h` uses `t0`, `t1` and `a0`. How many registers must it save?
3. Function `k` uses `s1` and `s2` and calls other functions. What must its prologue save?

<details><summary>Answers</summary>

1. `printf` may overwrite `t1` (caller-saved), so the counter gets corrupted. Fix: keep the counter in a callee-saved register (`s1`, saved once in the prologue), or save/restore `t1` around every call (slower).
2. **Zero.** All of them are caller-saved, and `h` calls nothing, so not even `ra` needs saving.
3. `ra` (it makes calls) + `s1` + `s2` (callee-saved registers it uses) = **3 registers**, restored in the epilogue.
</details>

### 2.9 ARM-specific points [QCOM][APPLE][TI]
- **AArch64:** 31 GPRs (x0–x30) + zero/SP register, fixed 32-bit instructions, load/store pair (`LDP/STP`), no general conditional execution (only `CSEL` etc.).
- **AArch32 (A32):** almost every instruction conditional (predication) — removed in A64 because it complicates OoO.
- **Thumb / Thumb-2:** 16/32-bit mixed encoding → ~30% smaller code, important for microcontrollers (Cortex-M runs *only* Thumb).
- **SIMD:** NEON (128-bit), **SVE/SVE2** (vector-length agnostic, 128–2048 bits).
- Weakly-ordered memory model (unlike x86 TSO) — relevant for lock-free code.

### 2.10 Translating C to assembly (RISC-V)
```c
// long sum(long *a, long n) { long s = 0; for (long i = 0; i < n; i++) s += a[i]; return s; }
sum:    li   t0, 0          # s = 0
        li   t1, 0          # i = 0
loop:   bge  t1, a1, done   # if i >= n exit
        slli t2, t1, 3      # t2 = i * 8
        add  t2, a0, t2     # &a[i]
        ld   t3, 0(t2)      # a[i]
        add  t0, t0, t3     # s += a[i]
        addi t1, t1, 1      # i++
        j    loop
done:   mv   a0, t0
        ret
```
An optimising compiler would use **pointer increment** (strength reduction) instead of recomputing `i*8`, and might unroll/vectorise.

---

## 3. Worked Numericals

**N1. Encoding capacity.** 32-bit instructions, 3 register operands, 64 registers. Register fields = 3 × 6 = 18 bits ⇒ opcode bits = 14 ⇒ up to **2¹⁴ = 16,384** three-register instructions.

**N2. Expanding opcode (GATE classic).** 16-bit instructions, 6-bit address fields. There are 14 two-address instructions. Max one-address instructions?
- Two-address: 12 bits addresses ⇒ 4-bit opcode ⇒ 16 patterns, 14 used ⇒ **2 spare patterns**.
- Each spare pattern extends with 6 more opcode bits ⇒ 2 × 2⁶ = **128 one-address instructions**.
- (If additionally zero-address instructions were needed, you'd reserve some of those 128 patterns.)

**N3. Endianness.** A little-endian machine stores `int x = 0x0A0B0C0D` at 0x200. What does `*(uint16_t*)0x202` read? Bytes: 200:0D, 201:0C, 202:0B, 203:0A → reads bytes (0B, 0A) little-endian ⇒ **0x0A0B**.

**N4. Struct size.** `struct S { char c; double d; short s; int i; };` (x86-64)
c@0, pad 7, d@8, s@16, pad 2, i@20 → end 24, align 8 ⇒ **24 bytes**. Reordered `{double d; int i; short s; char c;}` ⇒ 8+4+2+1 = 15 → **16 bytes**.

**N5. Branch range.** RISC-V B-type has a 13-bit signed offset (12 bits stored, LSB implicitly 0) ⇒ range **±4 KiB**. JAL: 21-bit ⇒ **±1 MiB**.

**N6. Caller-saved vs callee-saved cost.** Function `f` runs a loop that calls `g` **100 times**. Across each call, `f` needs **3 values** to survive (`i`, a pointer, a running sum).
- Keep them in **caller-saved** registers: 3 stores before + 3 loads after **each** call = 6 × 100 = **600 memory operations**.
- Keep them in **callee-saved** registers: `f` saves the 3 registers once in its prologue and restores them once in its epilogue = **6 memory operations**. (`g`, if it is a leaf using only temporaries, saves nothing.)
⇒ **100× fewer** save/restore operations. This is exactly why compilers put loop-carried values in callee-saved registers.

---

## 4. Interview Questions

### Basic
- **Q: ISA vs microarchitecture?** Contract vs implementation; same ISA, many microarchitectures (e.g., ARMv8: Cortex-A55 in-order, Apple Firestorm 8-wide OoO).
- **Q: What is endianness? Write code to detect it.** (See 2.6.)
- **Q: Why is a load-store architecture easier to pipeline?** Each instruction does at most one memory access in a fixed stage; ALU ops only read registers → uniform, simple pipeline.

### Intermediate
- **Q: RISC vs CISC — which is better today?** Neither inherently; x86 translates to µops; RISC gives simpler/wider decode and lower power; CISC has denser code. Implementation quality dominates.
- **Q [TI]: Why does Cortex-M use Thumb?** Code density → smaller flash, cheaper MCU, better I-cache use.
- **Q: Why does RISC-V not have condition flags?** Flags are an implicit shared register that creates false dependencies and complicates renaming/OoO; compare-and-branch is cleaner.
- **Q [DESHAW]: Caller-saved vs callee-saved — why have both?** Trade-off: short-lived values in caller-saved (no save if not live across a call), long-lived values in callee-saved (saved once by callee only if it uses them). See [2.8.1](#281-caller-saved-vs-callee-saved-registers-detailed-explanation) and N6.
- **Q [TI][QCOM]: Why must an interrupt handler save even caller-saved registers?** The interrupted code didn't make a call, so it didn't save its caller-saved registers. Cortex-M hardware stacks R0–R3, R12, LR, PC and xPSR automatically, which lets the ISR be a normal C function.

### Advanced / killer follow-ups
- **Q [APPLE]: Why can Apple decode 8+ instructions per cycle but x86 struggles?** Fixed 4-byte instructions ⇒ instruction boundaries known in advance; x86's 1–15-byte variable length requires finding boundaries serially (length decoding), hence µop caches.
- **Q: What is position-independent code and how does the ISA support it?** PC-relative addressing (`auipc`, RIP-relative) + GOT/PLT; needed for shared libraries and ASLR.
- **Q [QCOM]: Why did ARMv8 drop conditional execution on every instruction?** Predicated instructions add a flags dependency to every instruction, hurting OoO/renaming; branch predictors became good enough; kept `CSEL` for short branchless sequences.
- **Q [TI]: What's special about DSP addressing modes?** Circular buffers (modulo) for FIR filters without bounds checks; bit-reversed for FFT; zero-overhead loops.

---

## 5. Common Traps
- Saying "x86 is CISC so it's slower" — outdated.
- Mixing up which byte goes where in little-endian (LSB at lowest address).
- Forgetting struct tail padding (array of structs must keep each element aligned).
- Thinking endianness affects bit order within a byte or register arithmetic — it doesn't.
- Thinking "callee-saved" means the callee always saves it. It saves the register **only if it uses it**. Likewise the caller saves a caller-saved register **only if it needs the value after the call**.
- Forgetting that a non-leaf function must save `ra`/LR.

---

## 6. Revision Summary
- ISA = contract (spec); microarch = implementation (design); processor = physical chip (product). E.g., ARMv8-A → Firestorm → Apple M1.
- Load-store RISC: fixed length, simple decode, many registers. x86: variable length → µops.
- RISC-V formats: R, I, S, B, U, J; register fields fixed position.
- Little-endian: LSB at lowest address (x86, ARM, RISC-V). Network = big.
- Struct padding: members aligned to own size; sort large → small.
- Caller-saved (scratch: caller saves around a call if still needed) vs callee-saved (preserved: callee saves in prologue if it uses it). Temporaries → caller-saved; values live across calls → callee-saved. Leaf functions are cheap. ISRs must save everything.
- ARM: AArch64 31 GPRs, Thumb for density, NEON/SVE, weak memory model.
- DSP extras: circular & bit-reversed addressing, zero-overhead loops.
