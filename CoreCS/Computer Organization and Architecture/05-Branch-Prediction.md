# 05. Branch Prediction

> **Asked at:** [APPLE] [NVIDIA] [QCOM] predictor design (2-bit, gshare, tournament, TAGE, BTB, RAS) · [HFT] [DESHAW] "why is processing a sorted array faster?", branchless code, `likely/unlikely` · Spectre awareness.
> **Prerequisites:** [04 Pipelining](04-Pipelining-and-Hazards.md).

---

## 1. Intuition
A modern core fetches instructions ~15–20 cycles before it knows whether a branch is taken. Rather than wait, it **guesses and runs ahead speculatively**; if wrong, it throws the work away. Since ~1 in 5 instructions is a branch and a 6-wide core has ~100 instructions in flight, prediction accuracy of 95% vs 99% is the difference between a fast and a slow chip. Predictors work because **branch behaviour is highly repetitive** — per branch (loops) and across branches (correlation).

---

## 2. Deep Dive

### 2.1 What must be predicted (in the IF stage, before decode!)
1. **Is this instruction a branch?** — BTB hit tells you.
2. **Direction** (taken / not taken) — direction predictor.
3. **Target address** — BTB (direct), indirect predictor, Return Address Stack (returns).

### 2.2 Static prediction
| Scheme | Rule | Accuracy |
|---|---|---|
| Always not-taken | fall through | ~30–40% |
| Always taken | | ~60–70% |
| **BTFN** | backward-taken (loops), forward-not-taken | ~65–75% |
| Profile-guided / hints | compiler sets layout (`__builtin_expect`, `[[likely]]`) | good for biased branches |
Compiler hints mainly affect **code layout** (keep hot path fall-through, cold code out of the I-cache), not the hardware predictor on most modern CPUs.

### 2.3 1-bit predictor (Branch History Table)
Table of 1-bit entries indexed by low PC bits: "last time it was taken". Problem: a loop branch mispredicts **twice** per loop execution — at the exit and again at the next entry.

### 2.4 2-bit saturating counter — the workhorse
```
  Outcome T (taken) moves RIGHT, outcome N (not taken) moves LEFT; saturates at the ends.

        ──T──►        ──T──►        ──T──►
  [00]          [01]          [10]          [11]
 Strong NT     Weak NT       Weak T       Strong T
        ◄──N──        ◄──N──        ◄──N──
  N at 00 stays 00                    T at 11 stays 11

  predict NOT TAKEN: 00, 01      predict TAKEN: 10, 11   (prediction = MSB)
```
Needs **two wrong outcomes in a row** to flip the prediction ⇒ loop exit costs only one mispredict.

**Aliasing:** different branches whose PCs map to the same entry interfere (destructive aliasing) — bigger tables, tags, or hashing reduce it.

### 2.5 Correlating / two-level predictors
Branches correlate with **other recent branches**:
```c
if (x == 0) a = 1;      // B1
if (y == 0) b = 1;      // B2
if (x != y) ...         // B3 outcome depends on B1, B2
```
- **Global History Register (GHR):** shift register of the last h branch outcomes (all branches).
- **Local history:** per-branch history (captures patterns like T T T N in a loop of 4).
- **(m, n) predictor:** m bits of global history select among 2ᵐ n-bit counters per branch.
- **gshare:** index = PC ⊕ GHR → one table of 2-bit counters. Cheap, spreads entries, captures correlation.
```
     PC[low bits] ──┐
                    XOR ──► index ──► [ 2-bit counter table ] ──► predict
     GHR (h bits) ──┘
```
- **Tournament (Alpha 21264):** a local predictor + a global predictor + a **chooser** (2-bit counters that learn which one to trust per branch).
- **Perceptron predictor (AMD Zen uses neural-inspired designs):** weights × history bits; learns long linear correlations.
- **TAGE (state of the art, in most high-end cores today):** a base bimodal table + several **tagged** tables indexed with **geometrically increasing history lengths** (e.g., 5, 15, 44, 130 bits). Use the longest-history table that hits (tag match). Captures both short and very long correlations; accuracy commonly > 97–99%.

### 2.6 Branch Target Buffer (BTB)
A cache indexed by fetch PC: `{tag, predicted target, (type, counter)}`. Looked up **in IF**, in parallel with the I-cache, so the next fetch can be redirected with zero bubbles if it hits.
```
 PC ──► BTB ──hit──► next PC = predicted target (if predicted taken)
            └miss──► next PC = PC + fetch width
```
- **Return Address Stack (RAS):** on `call` push return address, on `ret` pop. Near-perfect for returns (a BTB would fail because one `ret` returns to many places). Overflow on deep recursion → mispredicts.
- **Indirect branch predictor** (virtual calls, switch jump tables, interpreters): uses history to pick among multiple targets (ITTAGE). C++ virtual calls in hot paths [HFT] are costly mainly when *polymorphic at that call site*.

### 2.7 Cost of misprediction
```
Branch CPI = branch fraction × misprediction rate × penalty
```
Modern penalty ≈ 15–20 cycles (x86, Apple ~13+). On a 4–8-wide core, that's **60–160 instruction slots** lost per mispredict.

### 2.8 Software side [HFT][DESHAW][NVIDIA]
**Sorted vs unsorted array (famous):**
```c
for (i = 0; i < N; i++)
    if (data[i] >= 128) sum += data[i];   // random data: ~50% mispredict → slow
                                          // sorted data: predictable → 3–6× faster
```
Fixes:
- **Branchless:** `sum += data[i] & -(data[i] >= 128);` or let the compiler emit `cmov`/`csel`.
- Compilers may vectorise it (SIMD compare + mask) — no branch at all.
- Predicated / branchless code always pays for both paths: good for **unpredictable** branches, bad for highly predictable ones (adds a data dependence).
- `__builtin_expect` / `[[likely]]` → layout; also put error handling `[[unlikely]]` / cold functions out of the hot path.
- Remove branches from hot loops: lookup tables, bit tricks, sentinels, loop unswitching.
- **GPU** analogue: warp divergence (see [12](12-GPU-Architecture.md)) — both paths execute serially with masks.

### 2.9 Speculation and security (Spectre v1)
The CPU executes past a mispredicted bounds check, loads secret-dependent data into the cache, then squashes — but the **cache state remains**, observable by timing (side channel). Mitigations: `lfence` after checks, index masking, retpolines (for indirect branches, Spectre v2), hardware fixes. Good "senior" talking point at Apple/Qualcomm/NVIDIA.

---

## 3. Worked Numericals

**N1. Loop of 10 iterations (pattern T×9, N×1, repeated many times).**
- 1-bit: mispredict at exit (predicted T, actual N) and at the first iteration of the next run (predicted N, actual T) ⇒ 2/10 wrong ⇒ **80%** accurate.
- 2-bit (in steady state at 11): exit → mispredict, state 10 (still predicts T); next entry correct ⇒ 1/10 wrong ⇒ **90%**.

**N2. Alternating T, N, T, N … with a 2-bit counter.**
- Start at 01 (weak N): T→miss→10; N→ predicts T, miss→01; T→miss … ⇒ **0% accuracy!**
- Start at 11: T hit→11; N miss→10; T hit→11; N miss … ⇒ **50%**.
- gshare/local predictor with ≥ 1 bit of history: "last was T → predict N" ⇒ **~100%** after warm-up. *Lesson: history captures patterns counters can't.*

**N3. Nested loop, 2-bit counters, inner loop of 4 iterations (T T T N), outer loop runs 1000 times.**
Inner branch (steady state): mispredicts only at exit ⇒ 1/4 ⇒ **75%** accurate. A local predictor with 4-bit history learns TTTN ⇒ ~100%.

**N4. CPI impact.** Base CPI 0.5 (IPC 2), 20% branches, penalty 15.
- 95% accuracy: CPI = 0.5 + 0.2 × 0.05 × 15 = **0.65** (30% slowdown).
- 98% accuracy: CPI = 0.5 + 0.2 × 0.02 × 15 = **0.56**.
Going from 95% to 98% gives a **16% speedup** — that's why companies spend so much area on predictors.

**N5. gshare indexing.** 4K-entry table (12-bit index), PC = 0x0040_1A3C, GHR (12 bits) = 0xA5F.
Index = PC[13:2] ⊕ GHR. PC >> 2 = 0x10068F; low 12 bits = 0x68F. 0x68F ⊕ 0xA5F = **0xCD0**.

---

## 4. Interview Questions

### Basic
- **Q: Why do we need branch prediction?** Pipeline must fetch the next instruction every cycle but branch outcome is known many cycles later; stalling would waste branch-frequency × penalty CPI.
- **Q: Draw the 2-bit saturating counter. Why is it better than 1-bit?** (See 2.4 and N1.)
- **Q: What is a BTB?** Cache of branch PC → target, consulted in IF to redirect fetch with no bubble.

### Intermediate
- **Q: What is a global history register, and what is gshare?** Shift register of recent outcomes; gshare XORs it with PC to index counters → captures correlation, reduces aliasing.
- **Q: How are function returns predicted?** Return Address Stack.
- **Q [HFT]: Why is processing a sorted array faster?** Branch becomes predictable (long runs of T then N). Fix for unsorted: branchless code / SIMD.
- **Q: When is branchless code slower?** When the branch is highly predictable — branchless adds a data dependence and executes both sides; the predictor would have made the branch free.

### Advanced / killer follow-ups
- **Q [APPLE]: Explain TAGE.** Multiple tagged tables with geometric history lengths; longest-matching-history provides the prediction; usefulness counters manage replacement; base bimodal fallback.
- **Q: How do you update the predictor when speculative branches are in flight?** Speculatively update the GHR at prediction time, checkpoint it per branch, restore on mispredict; update counters at commit/resolution.
- **Q: How would you predict indirect branches in an interpreter loop?** Indirect target predictor indexed with path/global history (ITTAGE); software: threaded code / computed goto gives each opcode its own dispatch branch.
- **Q [NVIDIA]: Do GPUs have branch predictors?** Essentially no — they hide latency with massive multithreading (switch warps) instead of speculating; divergence handled by masking/reconvergence.
- **Q: Explain Spectre v1.** (See 2.9.)

---

## 5. Common Traps
- Confusing direction prediction with target prediction (BTB).
- Claiming `__builtin_expect` "trains the predictor" — mostly code layout.
- Forgetting the 2-bit counter's initial state matters for short traces.
- Assuming branchless is always better.

---

## 6. Revision Summary
- Predict: is-branch + direction + target, all in IF.
- Static: BTFN. Dynamic: 1-bit (2 misses/loop), 2-bit saturating (1 miss/loop).
- Correlation: GHR, local history, gshare = PC ⊕ GHR, tournament = local + global + chooser, TAGE = geometric histories + tags.
- BTB for targets, RAS for returns, ITTAGE for indirect.
- CPI cost = freq × miss rate × penalty (~15–20 cycles today).
- Software: sort/partition, branchless (cmov), SIMD, layout hints, avoid polymorphic virtual calls in hot path.
- GPUs don't predict; they multithread. Spectre = speculation + cache side channel.
