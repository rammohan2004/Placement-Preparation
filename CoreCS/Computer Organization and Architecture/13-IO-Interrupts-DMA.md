# 13. I/O, Interrupts, DMA & Buses (Embedded focus)

> **Asked at:** **[TI] [QCOM]** (embedded SW, firmware, SoC, validation roles) — MMIO & `volatile`, ISR design, interrupt latency, DMA + cache coherence, I2C vs SPI vs UART, AMBA buses · [APPLE] SoC basics · [HFT] polling vs interrupts, kernel bypass, PCIe.
> **Prerequisites:** [06 Cache](06-Cache-Memory.md), [10 Consistency](10-Cache-Coherence-and-Memory-Consistency.md).

---

## 1. Intuition
The CPU talks to devices (UART, ADC, NIC, SSD, GPU) by reading/writing **device registers**. The question is *who waits for whom*:
- **Polling:** CPU keeps asking "are you done?" — simple, wastes cycles (but lowest latency!).
- **Interrupts:** device taps the CPU on the shoulder when it needs attention.
- **DMA:** a helper engine moves the bulk data itself and interrupts the CPU only when the whole job is done.

---

## 2. Deep Dive

### 2.1 Addressing devices
| | Port-mapped I/O (PMIO) | **Memory-mapped I/O (MMIO)** |
|---|---|---|
| Access | special instructions (`in`/`out` on x86) | ordinary loads/stores to reserved addresses |
| Address space | separate | shared with memory |
| Used by | legacy x86 | ARM, RISC-V, all MCUs/SoCs, PCIe BARs |
```c
#define UART0_BASE   0x4000C000u
#define UART_DR      (*(volatile uint32_t *)(UART0_BASE + 0x00))   // data register
#define UART_FR      (*(volatile uint32_t *)(UART0_BASE + 0x18))   // flag register
#define UART_FR_TXFF (1u << 5)                                     // TX FIFO full

void uart_putc(char c) {
    while (UART_FR & UART_FR_TXFF) { }   // poll
    UART_DR = (uint32_t)c;
}
```
**Why `volatile`?** Without it the compiler may cache `UART_FR` in a register and spin forever, or merge/elide writes. `volatile` forces every access to actually happen, in program order *relative to other volatile accesses* (compiler only).
**Also needed:** MMIO regions are mapped **uncached / device memory** (MPU/MMU attributes) so the cache doesn't hold stale register values; on ARM, device memory is ordered, but ordering with normal memory (e.g., DMA buffers) needs **`DMB`/`DSB`** barriers.

### 2.2 Polling vs interrupts vs DMA
| | Polling | Interrupt-driven | DMA |
|---|---|---|---|
| CPU cost | high when waiting | per event (~100s–1000s cycles incl. context save) | setup + 1 interrupt per block |
| Latency | **lowest, deterministic** | ISR entry latency | per block |
| Best for | fast devices with frequent data, or ultra-low latency (**HFT busy-polling NICs**, DPDK) | infrequent, unpredictable events (buttons, low-rate UART) | bulk transfers (ADC streams, audio, SPI flash, network, disk) |

### 2.3 Interrupt handling — step by step
1. Device asserts an IRQ line → interrupt controller (ARM **NVIC** on Cortex-M, **GIC** on Cortex-A, x86 **APIC**) prioritises it.
2. CPU finishes / abandons the current instruction (precise), **saves context** (PC, status; Cortex-M hardware-stacks R0–R3, R12, LR, PC, xPSR automatically).
3. Looks up the handler in the **vector table**, switches mode/stack, jumps to the **ISR**.
4. ISR acknowledges/clears the device interrupt flag, does minimal work.
5. Return-from-interrupt restores context.

**Cortex-M specifics [TI][QCOM]:** vectored + **nested** (higher priority preempts lower), configurable priorities, **tail-chaining** (back-to-back ISRs skip unstack/restack), **late arrival** optimisation; entry latency ~12 cycles (M3/M4) with zero-wait-state memory.

**Interrupts vs exceptions vs traps:**
| | Source | Synchronous? | Example |
|---|---|---|---|
| Interrupt | external device | asynchronous | timer, UART RX |
| Exception / fault | the executing instruction | synchronous | divide-by-zero, page fault, bus fault |
| Trap / software interrupt | intentional instruction | synchronous | `syscall`, `svc`, `ecall` |

### 2.4 Writing good ISRs
- **Keep it short:** read data, clear flag, push to a ring buffer, set a flag; do heavy work in the main loop / a task (**top half / bottom half**, Linux softirq/tasklet/workqueue, RTOS deferred task).
- No blocking calls, no `printf`, no `malloc`, no floating point unless the context saves FPU state (Cortex-M4F lazy stacking).
- Shared variables between ISR and main: `volatile` + **atomic access** (disable interrupts briefly, or use LDREX/STREX, or single-word accesses that are atomic on that architecture).
- Clear the interrupt source **before** returning (else immediate re-entry) — and on some buses, read back to ensure the write landed (write buffers!).
- Beware **priority inversion** and missing **reentrancy**.
- **Interrupt latency** = time from IRQ assertion to first ISR instruction: hardware latency + longest interrupts-disabled section + higher-priority ISRs running.

```c
volatile uint8_t rx_buf[256];
volatile uint8_t head;                     // written by ISR only
uint8_t tail;                              // written by main only

void UART0_IRQHandler(void) {
    uint8_t c = (uint8_t)UART_DR;          // reading DR clears RX flag on many UARTs
    rx_buf[head] = c;
    head = (uint8_t)(head + 1);            // 8-bit index wraps at 256; single-byte write is atomic
}
int uart_getc(void) {
    if (tail == head) return -1;
    return rx_buf[tail++];
}
```
(SPSC ring buffer — same idea as [10 §3.5](10-Cache-Coherence-and-Memory-Consistency.md), with ISR as producer.)

### 2.5 DMA (Direct Memory Access)
The CPU programs a DMA controller with **source, destination, length, transfer width, increment mode, trigger** and starts it; the DMA engine becomes a **bus master** and moves data; an interrupt signals completion (or half-complete for **ping-pong/double buffering**).
- **Modes:** burst/block (hold the bus for the whole block — fast, starves the CPU), **cycle stealing** (one word at a time, interleaved), transparent (only when the bus is idle).
- **Scatter-gather / linked descriptors:** a chain of descriptors in memory lets one DMA job handle non-contiguous buffers (NICs, audio).
- **Peripheral-triggered DMA:** e.g., ADC conversion complete → DMA moves sample → after N samples, interrupt.
- **Double buffering:** CPU processes buffer A while DMA fills buffer B; swap on half/full-complete interrupts — standard for audio/DSP on TI C2000/C6000.

### 2.6 DMA & cache coherence ⭐ (top [TI]/[QCOM] question)
Non-coherent DMA (most MCUs/many SoCs) bypasses the CPU caches:
| Direction | Danger | Fix (before/after transfer) |
|---|---|---|
| CPU → device (TX) | data still dirty in CPU cache; DMA reads stale memory | **Clean (write back)** the buffer range before starting DMA |
| Device → CPU (RX) | CPU reads stale cached lines after DMA wrote memory | **Invalidate** the buffer range (before and/or after DMA), then read |
- DMA buffers must be **cache-line aligned and sized**: invalidating a line shared with other data destroys that data (or a dirty eviction overwrites DMA data).
- Alternatives: put DMA buffers in **non-cacheable** memory (MPU region), use **coherent DMA** (ACP / CCI / IO-coherent interconnects; PCIe on x86 is snooped/coherent).
- Linux: `dma_map_single` / `dma_sync_single_for_cpu/device` do exactly these clean/invalidate operations.

### 2.7 Buses and interconnects
**On-chip (ARM AMBA):**
| Bus | Use | Traits |
|---|---|---|
| **AXI** | high-performance (CPU, DDR, DMA, GPU) | separate read/write address & data channels, burst, **multiple outstanding transactions, out-of-order completion (IDs)** |
| **AHB** | mid-performance (MCU main bus) | pipelined, single master per transfer, bursts |
| **APB** | low-speed peripherals (UART, timers, GPIO) | simple, unpipelined, low power |
| ACE / CHI | coherent interconnects | coherence messages (snoop) |
AXI handshake: transfer happens when **VALID && READY** on a channel.

**Off-chip serial protocols [TI][QCOM]:**
| | UART | I²C | SPI |
|---|---|---|---|
| Wires | TX, RX (+GND) | SDA, SCL (open-drain + pull-ups) | SCLK, MOSI, MISO, CS per slave |
| Clock | asynchronous (agreed baud) | synchronous | synchronous |
| Topology | point-to-point | multi-master, multi-slave, 7/10-bit addresses | single master, many slaves via chip select |
| Duplex | full | half | full |
| Speed | ≤ ~1–5 Mbps typical (115200 common) | 100 k / 400 k / 1 M / 3.4 Mbps | 10s of MHz |
| Overhead | start + stop bits | address + ACK per byte | none |
| Use | debug consoles, GPS, BT modules | sensors, EEPROM, PMICs | flash, displays, ADCs |
I²C needs pull-ups because lines are **open-drain** (wired-AND) — this enables **clock stretching** and **arbitration**.

**PCIe [NVIDIA][HFT]:** serial point-to-point **lanes** (x1…x16), packet-based (TLPs), MMIO via BARs, MSI/MSI-X interrupts (in-band writes, no wires). Per-lane ≈ 2 (Gen3: ~1 GB/s), Gen4 ~2 GB/s, Gen5 ~4 GB/s per direction. Round-trip latency ~0.5–1 µs.

### 2.8 Networking I/O for HFT (bridge to OS)
Normal path: NIC DMA → ring buffer → interrupt → kernel TCP/IP stack → socket → copy to user (~5–20 µs).
Low-latency path: **kernel bypass** (Solarflare/AMD **OpenOnload/ef_vi**, **DPDK**, RDMA) — the NIC's rings are mapped into user space, the app **busy-polls**, zero copy, no interrupts, no syscalls ⇒ ~1–2 µs; FPGA NICs go sub-µs. Also: interrupt coalescing (throughput vs latency knob), NAPI (switch from interrupts to polling under load).

---

## 3. Worked Numericals

**N1. Polling overhead (P&H style).** 1 GHz CPU; one poll = 400 cycles.
- Mouse polled 30×/s: 12,000 cycles/s ⇒ **0.0012%** of CPU — fine.
- Disk at 4 MB/s, 16-byte transfers, polled continuously: 250,000 polls/s × 400 = 10⁸ cycles/s ⇒ **10%** of CPU, even when idle (if always polling).

**N2. DMA instead.** Same disk, DMA of 8 KB blocks; setup 1000 cycles + completion interrupt 500 cycles.
Blocks/s = 4 MB / 8 KB = 512 ⇒ 512 × 1500 = 768,000 cycles/s ⇒ **0.077%** of CPU (≈ 130× better than polling).

**N3. UART throughput.** 115200 baud, 8N1 (1 start + 8 data + 1 stop = 10 bits): 115200 / 10 = **11,520 bytes/s** ⇒ an interrupt-per-byte ISR fires every ~87 µs. With a 16-byte FIFO and an interrupt at 14 bytes ⇒ ~823 interrupts/s.

**N4. I²C effective rate.** 400 kHz fast mode; reading 1 data byte from a register = START + addr+W (9 bits) + reg (9) + repeated START + addr+R (9) + data (9) + STOP ≈ 38 bit-times ≈ **95 µs** ⇒ ~10.5 K register reads/s — protocol overhead matters.

**N5. Interrupt latency budget.** Cortex-M4 at 100 MHz: hardware entry 12 cycles (120 ns) + longest critical section with interrupts disabled 200 cycles (2 µs) ⇒ worst-case latency ≈ **2.12 µs** (+ any higher-priority ISR time). Shortening critical sections is the main lever.

**N6. DMA buffer alignment.** 64-byte cache lines; RX DMA buffer of 100 bytes at address 0x2000_0010. Lines touched: 0x2000_0000 – 0x2000_007F (2 lines), partially shared with neighbouring variables ⇒ invalidation may corrupt them. Fix: align to 64 and round size to **128 B**.

---

## 4. Interview Questions

### Basic
- **Q: Polling vs interrupts vs DMA — when to use which?** (2.2.)
- **Q: Memory-mapped vs port-mapped I/O?** (2.1.)
- **Q: What does `volatile` do? Give an embedded example.** Prevents compiler from caching/eliding accesses — MMIO registers, variables shared with ISRs, `setjmp`.
- **Q: UART vs SPI vs I²C?** (2.7 table.)

### Intermediate
- **Q: What happens when an interrupt occurs?** (2.3.)
- **Q: What should and shouldn't you do in an ISR?** (2.4.)
- **Q: How do you share data between an ISR and main code safely?** volatile + atomic access (disable IRQs briefly / exclusive access / lock-free SPSC ring buffer).
- **Q: What is interrupt latency and what affects it?** (2.4.)
- **Q: Why does I²C need pull-up resistors?** Open-drain lines; allows multiple devices to pull low (wired-AND), arbitration, clock stretching.
- **Q: What is tail-chaining in NVIC?** Back-to-back pending ISRs skip the unstack/stack sequence (~6 cycles instead of ~24).

### Advanced / killer follow-ups
- **Q [TI][QCOM]: DMA reads stale data / the CPU sees stale DMA data. Why and how do you fix it?** (2.6) clean before TX, invalidate for RX, align buffers, or non-cacheable/coherent memory.
- **Q: Is `volatile` sufficient for MMIO ordering on ARM?** Compiler ordering yes; hardware ordering between device and normal memory needs `DMB`/`DSB` (e.g., write DMA descriptor in normal memory, `DSB`, then ring doorbell register).
- **Q: Design a double-buffered ADC acquisition with DMA.** Circular DMA over buffer[2][N], half-complete IRQ → process half 0, complete IRQ → process half 1; ensure processing time < N / sample rate; invalidate cache for the half before processing.
- **Q [HFT]: Why do trading systems busy-poll instead of using interrupts?** Interrupt + context switch + scheduler wakeup cost microseconds and add jitter; a dedicated isolated core spinning on the NIC ring gives the lowest, most consistent latency.
- **Q: What is MSI-X?** Interrupts as memory writes over PCIe, many vectors per device ⇒ per-queue interrupts steered to specific cores.
- **Q: What is priority inversion? Fix?** Low-priority task holds a resource needed by high-priority task while medium preempts the low one; fix with priority inheritance / ceiling (Mars Pathfinder story).

---

## 5. Common Traps
- Saying `volatile` makes code thread-safe or atomic.
- Forgetting to clear the interrupt flag → ISR storm.
- DMA buffers on the stack (may go out of scope) or unaligned to cache lines.
- Assuming DMA is always faster for tiny transfers (setup overhead).

---

## 6. Revision Summary
- MMIO + `volatile` + uncached device memory + barriers (DMB/DSB).
- Polling (lowest latency, burns CPU) · interrupts (event-driven) · DMA (bulk).
- Interrupt flow: assert → prioritise → save context → vector → ISR → clear flag → return.
- Cortex-M NVIC: nested, vectored, tail-chaining, ~12-cycle entry.
- ISR: short, no blocking, defer work, volatile + atomic sharing.
- DMA coherence: clean before TX, invalidate for RX, align to cache lines.
- AXI (high perf, outstanding/OoO) · AHB · APB (low speed).
- UART async 2-wire · I²C 2-wire open-drain addressed half-duplex · SPI 4-wire full-duplex fast.
- HFT: kernel bypass + busy polling.
