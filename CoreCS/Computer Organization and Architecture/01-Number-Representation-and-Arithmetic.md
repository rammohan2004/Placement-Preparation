# 01. Number Representation & Computer Arithmetic

> **Asked at:** [TI] [QCOM] fixed-point & overflow (DSP firmware) · [NVIDIA] FP16/BF16/FP8 formats for AI · [HFT] [DESHAW] floating-point traps, integer overflow, bit tricks · [APPLE] [NVIDIA-HW] adder/multiplier design.
> **Prerequisites:** binary, boolean algebra.

---

## 1. Intuition (the 30-second version)
A computer only stores **fixed-width bit patterns**. "Number representation" is the *agreement* on what a pattern means. Every interview trap here comes from one fact: **fixed width ⇒ finite set of values ⇒ overflow, rounding, and wrap-around.** Integers wrap like a car odometer; floats are like scientific notation with a fixed number of significant digits.

---

## 2. Deep Dive

### 2.1 Unsigned and signed integers
For n bits:

| Representation | Range | Zero(s) | Used today? |
|---|---|---|---|
| Unsigned | 0 … 2ⁿ − 1 | one | yes |
| Sign-magnitude | −(2ⁿ⁻¹−1) … 2ⁿ⁻¹−1 | +0, −0 | FP significand only |
| 1's complement | −(2ⁿ⁻¹−1) … 2ⁿ⁻¹−1 | +0, −0 | Internet checksum |
| **2's complement** | **−2ⁿ⁻¹ … 2ⁿ⁻¹−1** | one | **all modern CPUs** |

**Why 2's complement wins:** the *same adder circuit* adds signed and unsigned numbers; subtraction is `a + (~b) + 1` (just set carry-in = 1); only one zero.

Value of a 2's complement number: the MSB has **negative weight**.
```
b(n-1) b(n-2) ... b0  =  -b(n-1)·2^(n-1) + Σ b(i)·2^i
1011 (4-bit) = -8 + 0 + 2 + 1 = -5
```
- Negation: `-x = ~x + 1`.
- **Asymmetry:** −2ⁿ⁻¹ has no positive counterpart. `-INT_MIN == INT_MIN`; `abs(INT_MIN)` overflows (UB in C).
- **Sign extension:** copy the MSB into new high bits (`1011` → `11111011` = −5 still). Zero extension is for unsigned.

### 2.2 Overflow vs carry — the classic confusion
| Flag | Meaning | Detect |
|---|---|---|
| **Carry (C)** | Unsigned result doesn't fit | carry out of MSB |
| **Overflow (V)** | Signed result doesn't fit | carry *into* MSB ≠ carry *out of* MSB, i.e. both operands same sign and result sign differs |

```
  0111 (+7)          1000 (-8)
+ 0001 (+1)        + 1111 (-1)
= 1000 (-8)  V=1   = 0111 (+7)  V=1, C=1
```
Adding a positive and a negative number **can never overflow**.

> **C/C++ trap [HFT][DESHAW]:** signed overflow is *undefined behaviour*, unsigned overflow wraps mod 2ⁿ. The compiler may delete your `if (x + 1 < x)` check. Use `__builtin_add_overflow` or check before adding.

### 2.3 Fixed-point (Q format) — [TI] [QCOM] favourite
Integer hardware, implied binary point. **Qm.n**: m integer bits (excl. sign), n fractional bits.
- **Q15** (16-bit): 1 sign + 15 fraction bits. Range [−1, 1 − 2⁻¹⁵], resolution 2⁻¹⁵ ≈ 3.05×10⁻⁵.
- Real value = integer / 2ⁿ. So 0.5 in Q15 = 0x4000 = 16384.
- **Multiply:** Q15 × Q15 = Q30 in 32 bits → shift right 15 to return to Q15 (add 2¹⁴ before shifting to round).
- The only overflow case for Q15×Q15: (−1)×(−1) = +1, not representable → **saturate** to 0x7FFF.
- **Saturating arithmetic:** clamp to max/min instead of wrapping. DSPs (TI C6000, Qualcomm Hexagon) and ARM (`QADD`) have instructions for it — wrap-around in audio = loud click.

```c
int16_t q15_mul(int16_t a, int16_t b) {
    int32_t p = (int32_t)a * b;      // Q30
    p += 1 << 14;                    // round to nearest
    p >>= 15;                        // back to Q15
    if (p > 32767)  p = 32767;       // saturate (only -1*-1)
    return (int16_t)p;
}
```
Why fixed-point? No FPU on small MCUs / lower power / deterministic timing.

### 2.4 IEEE-754 floating point
```
 single (32): | S(1) | Exponent(8), bias 127   | Fraction(23) |
 double (64): | S(1) | Exponent(11), bias 1023 | Fraction(52) |
 value (normal) = (-1)^S × 1.F × 2^(E - bias)
```

| E field | F | Meaning |
|---|---|---|
| 0 | 0 | ±0 |
| 0 | ≠0 | **Subnormal**: (−1)ˢ × 0.F × 2^(1−bias) — gradual underflow |
| 1 … max−1 | any | Normal |
| all 1s | 0 | ±∞ |
| all 1s | ≠0 | NaN |

**Key numbers (float32):**
| Quantity | Value |
|---|---|
| Largest normal | (2 − 2⁻²³) × 2¹²⁷ ≈ 3.4 × 10³⁸ |
| Smallest normal | 2⁻¹²⁶ ≈ 1.18 × 10⁻³⁸ |
| Smallest subnormal | 2⁻¹⁴⁹ ≈ 1.4 × 10⁻⁴⁵ |
| Machine epsilon (gap after 1.0) | 2⁻²³ ≈ 1.19 × 10⁻⁷ |
| Decimal precision | ~7 digits (double: ~16) |
| Largest consecutive exact integer | 2²⁴ = 16,777,216 (double: 2⁵³) |

**Why a bias instead of 2's complement exponent?** So that bigger floats have bigger bit patterns — positive floats can be **compared as integers** (fast sorting/comparison hardware).

**Spacing grows with magnitude:** between 2ᵏ and 2ᵏ⁺¹ there are always 2²³ floats, so the gap is 2ᵏ⁻²³. At 2²⁴, gap = 2 → `16777216.0f + 1.0f == 16777216.0f`.

**Rounding:** default is *round to nearest, ties to even*. Uses Guard, Round, Sticky bits in hardware.

**Properties that bite in interviews:**
- Not associative: `(a + b) + c ≠ a + (b + c)` (e.g. 1e20 + −1e20 + 1). ⇒ parallel reductions (GPU!) give run-to-run differences.
- `0.1 + 0.2 != 0.3` — 0.1 is a repeating binary fraction (0.000110011…).
- `NaN != NaN` (only value not equal to itself — `x != x` is a NaN test). `+0.0 == -0.0` but `1/+0 = +∞`, `1/−0 = −∞`.
- **Subnormals can be 10–100× slower** on many x86 CPUs (microcode assist). HFT/graphics set **FTZ/DAZ** (flush-to-zero / denormals-are-zero).
- **FMA** (fused multiply-add) computes a×b+c with one rounding — more accurate and the basis of GPU/tensor throughput numbers.
- Compare with tolerance: `fabs(a-b) <= eps * max(fabs(a), fabs(b))`.
- Summing many floats: sort small-to-large, pairwise sum, or **Kahan summation**.

### 2.5 Reduced-precision formats — [NVIDIA] [QCOM] [APPLE] (AI accelerators)
| Format | S/E/M bits | Max | Why it exists |
|---|---|---|---|
| FP32 | 1/8/23 | 3.4e38 | baseline |
| FP16 (half) | 1/5/10 | 65,504 | 2× throughput, but small range → needs loss scaling in training |
| **BF16** | 1/8/7 | ~3.4e38 | same range as FP32 (just truncate FP32!), less precision — training friendly |
| TF32 (NVIDIA) | 1/8/10 | ~3.4e38 | Tensor-core internal format, FP32 range + FP16 precision |
| FP8 E4M3 | 1/4/3 | 448 | inference / forward pass |
| FP8 E5M2 | 1/5/2 | 57,344 | gradients (need range) |
| INT8 / INT4 | — | — | quantized inference |

Interview line: *"Deep learning cares more about range than precision, which is why BF16 keeps the 8-bit exponent of FP32."*

### 2.6 Adders — [NVIDIA-HW] [APPLE-HW] [QCOM-HW] [TI]
**Full adder:** `S = a ⊕ b ⊕ cin`, `cout = ab + cin(a ⊕ b)`.

| Adder | Delay | Area | Idea |
|---|---|---|---|
| Ripple-carry (RCA) | O(n) | O(n) | chain full adders |
| Carry-lookahead (CLA) | O(log n) | O(n log n) | compute carries in parallel |
| Carry-select | O(√n) | ~2× RCA | compute both carry-in cases, mux |
| Carry-skip | O(√n) | small | skip blocks whose propagate = 1 |
| Carry-save (CSA) | O(1) per level | O(n) | don't propagate; output sum + carry vectors (used in multipliers) |
| Kogge-Stone / Brent-Kung | O(log n) | KS: big & fast; BK: smaller, ~2 log n levels | parallel-prefix CLAs used in real CPUs |

**CLA equations:**
```
Generate  g_i = a_i · b_i        (this bit creates a carry)
Propagate p_i = a_i ⊕ b_i        (this bit passes an incoming carry)
c_{i+1}   = g_i + p_i · c_i
c1 = g0 + p0c0
c2 = g1 + p1g0 + p1p0c0
c3 = g2 + p2g1 + p2p1g0 + p2p1p0c0      ← all carries in 2 gate levels (fan-in grows!)
```
Fan-in growth is why real designs use **hierarchical/prefix** CLAs with group (G, P) signals:
`(G, P)_{i:j} = (G_hi + P_hi·G_lo, P_hi·P_lo)` — an *associative* operator ⇒ log-depth prefix tree.

### 2.7 Multipliers
1. **Shift-and-add** (sequential): n cycles for n-bit, one partial product per cycle.
2. **Booth's algorithm (radix-2):** recode multiplier by looking at pairs (Qᵢ, Qᵢ₋₁):

| Qᵢ Qᵢ₋₁ | Action |
|---|---|
| 0 0 / 1 1 | none (inside a run) |
| 0 1 | A ← A + M (end of a run of 1s) |
| 1 0 | A ← A − M (start of a run of 1s) |
Then arithmetic-shift-right (A, Q, Q₋₁). Handles signed numbers natively; good for runs of 1s.

3. **Radix-4 (modified) Booth:** looks at 3 bits, digits ∈ {−2,−1,0,1,2} ⇒ **halves the number of partial products**. Used in real hardware.
4. **Wallace / Dadda tree:** reduce all partial products with layers of carry-save adders (3:2 compressors) in O(log n) levels, then one fast CPA at the end.

### 2.8 Division (brief)
- **Restoring:** subtract divisor; if negative, restore (add back) and set quotient bit 0.
- **Non-restoring:** don't restore; if remainder negative, *add* next step instead of subtract. One add/sub per step.
- **SRT** (radix-4 SRT used in CPUs — the 1994 Pentium FDIV bug was missing entries in an SRT lookup table).
- Division is slow (~20–40+ cycles for 64-bit int on x86; multiply ~3). **Compilers turn division by constants into multiply + shift** ("magic numbers") — HFT folklore.

### 2.9 Bit-manipulation toolkit [HFT][DESHAW][QCOM][TI]
| Trick | Expression |
|---|---|
| Is power of two | `x && !(x & (x-1))` |
| Clear lowest set bit | `x & (x-1)` |
| Isolate lowest set bit | `x & -x` |
| Popcount | `__builtin_popcount(x)` / loop `x &= x-1` |
| Set / clear / toggle bit k | `x \| (1<<k)`, `x & ~(1<<k)`, `x ^ (1<<k)` |
| Round up to multiple of 2ᵏ | `(x + (1<<k) - 1) & ~((1<<k) - 1)` |
| x mod 2ᵏ (unsigned) | `x & ((1<<k) - 1)` |
| Branchless abs (32-bit) | `m = x >> 31; (x ^ m) - m` |
| Branchless min | `y ^ ((x ^ y) & -(x < y))` |
| Sign of x | `(x > 0) - (x < 0)` |

Right shift of a negative signed int is implementation-defined before C++20 (arithmetic since C++20); `-7 >> 1 = -4` (rounds toward −∞) whereas `-7 / 2 = -3` (toward 0).

---

## 3. Worked Numericals

**N1. Encode −6.75 in IEEE-754 single precision.**
1. 6.75 = 110.11₂ = 1.1011 × 2²
2. S = 1; E = 2 + 127 = 129 = 1000 0001
3. F = 1011 000…0 (23 bits)
4. Bits: `1 10000001 10110000000000000000000` → `1100 0000 1101 1000 0000…` = **0xC0D80000**

**N2. Decode 0x41200000.**
`0100 0001 0010 0000 …` → S=0, E=1000 0010=130 → 2³, F=010… → 1.01₂ = 1.25 → 1.25 × 8 = **10.0**

**N3. Booth multiply 7 × (−3) with 4-bit registers.** M = 0111, −M = 1001, Q = 1101.

| Step | (Q₀,Q₋₁) | Action | A | Q | Q₋₁ |
|---|---|---|---|---|---|
| init | | | 0000 | 1101 | 0 |
| 1 | 10 | A−M → 1001; ASR | 1100 | 1110 | 1 |
| 2 | 01 | A+M → 0011; ASR | 0001 | 1111 | 0 |
| 3 | 10 | A−M → 1010; ASR | 1101 | 0111 | 1 |
| 4 | 11 | none; ASR | 1110 | 1011 | 1 |
Result A:Q = 1110 1011 = **−21** ✓

**N4. Q15:** multiply 0.75 × (−0.5). 0.75 = 24576 (0x6000), −0.5 = −16384 (0xC000). Product = −402,653,184 (Q30). >>15 → −12288 = −12288/32768 = **−0.375** ✓

**N5. How many distinct float32 values lie in [1, 2)?** All with E = 127: 2²³ = **8,388,608**. In [2, 4) also 2²³ — same count, double the spacing.

**N6. Delay comparison:** 64-bit RCA with 2 gate-delays per bit carry ≈ 128 gate delays; a radix-2 prefix (Kogge-Stone) adder needs log₂64 = 6 prefix levels + setup (p, g) + final XOR ≈ 6×2 + 2 ≈ **~14 gate delays** → ~9× faster.

---

## 4. Interview Questions

### Basic
- **Q: Why do computers use 2's complement?** Single zero, the same adder does signed/unsigned add, subtraction = add the inverted operand with carry-in 1, sign test = MSB.
- **Q: Range of an 8-bit signed integer?** −128 … 127.
- **Q: How do you detect signed overflow in hardware?** Carry into MSB XOR carry out of MSB (equivalently: operands same sign, result opposite sign).
- **Q: Why is `0.1 + 0.2 != 0.3`?** 0.1 and 0.2 aren't exactly representable in binary; their rounded sum rounds to a different double than 0.3's rounding.

### Intermediate
- **Q [TI]: What is Q15 and how do you multiply two Q15 numbers?** Explain Q30 intermediate, rounding, >>15, saturation for (−1)(−1).
- **Q: What are denormals and why might you disable them?** Gradual underflow near zero; slow (microcode assist) on many CPUs → FTZ/DAZ in latency-critical code; cost: lose precision near zero.
- **Q [NVIDIA]: FP16 vs BF16?** Same 16 bits; FP16 has 5-bit exponent (range 6e−5…65504, more precision), BF16 has 8-bit exponent (FP32 range, 3 significant digits). BF16 avoids overflow/underflow in training without loss scaling.
- **Q: Why is the exponent biased?** Lets non-negative floats be ordered as unsigned integers → cheap comparison.
- **Q: What's the largest integer a float can represent exactly such that all smaller integers are also exact?** 2²⁴ (double: 2⁵³). Relevant when storing prices/IDs as floats [HFT] — never store prices as float.

### Advanced / killer follow-ups
- **Q [APPLE-HW]: Why is a CLA faster and why don't we use a flat 64-bit CLA?** Carries computed in parallel from g/p; flat version has huge fan-in/area → use hierarchical / parallel-prefix (Kogge-Stone fast, big wiring; Brent-Kung fewer cells, more levels).
- **Q: How does radix-4 Booth help?** Halves the partial products (n/2); digits {−2…2} are trivial shifts/negations.
- **Q [HFT]: Prices are decimals — how would you represent them?** Fixed-point integers (e.g., price in 1e-8 units in int64) — exact, fast, deterministic; float is inexact and non-associative.
- **Q [DESHAW]: Is `a * b / c` safe for ints?** Intermediate overflow; widen to 64-bit or reorder; for signed it's UB.
- **Q: Can float addition give different results on CPU vs GPU for the same data?** Yes — reduction order differs (non-associativity), FMA contraction, FTZ settings.
- **Q: Why does division by a constant compile to a multiply?** x/d ≈ (x × ⌈2ᵏ/d⌉) >> k, exact for chosen k; multiply ~3 cycles vs divide ~20–40.

---

## 5. Common Traps
- Saying "overflow = carry out" — that's unsigned; signed uses V flag.
- Forgetting INT_MIN asymmetry (`abs`, negation, `INT_MIN / -1` traps on x86!).
- Thinking float has "7 decimal places" — it has ~7 *significant* digits, any magnitude.
- Using `==` on floats; using float for money.
- Mixing signed and unsigned in C comparisons: `-1 < 0u` is **false** (−1 converts to UINT_MAX).
- Forgetting saturation in DSP code.

---

## 6. Revision Summary
| Must remember | |
|---|---|
| n-bit 2's complement range | −2ⁿ⁻¹ … 2ⁿ⁻¹−1 |
| Signed overflow detect | Cin(MSB) ⊕ Cout(MSB) |
| float32 | 1/8/23, bias 127, ~7 digits, ε = 2⁻²³ |
| float64 | 1/11/52, bias 1023, ~16 digits, exact ints to 2⁵³ |
| BF16 vs FP16 | 1/8/7 (range) vs 1/5/10 (precision) |
| Special values | E=0 → zero/subnormal; E=all-ones → ∞/NaN |
| Q15 multiply | 32-bit product, +2¹⁴, >>15, saturate |
| Adders | RCA O(n) · CLA/prefix O(log n) · carry-select O(√n) · CSA O(1) |
| Multipliers | Booth radix-4 halves PPs; Wallace tree O(log n) |
| Bit tricks | `x&(x-1)`, `x&-x`, power-of-two test |
