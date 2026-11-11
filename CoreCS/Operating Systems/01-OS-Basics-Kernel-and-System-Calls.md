# 01. OS Basics, Kernel & System Calls

> **Asked at:** [AMZN] [DESHAW] "what is a system call / what happens when you call `printf`" · [HFT] cost of a syscall, vDSO, why the hot path avoids the kernel · [NVIDIA] [QCOM] user vs kernel mode, privilege levels, driver context · [APPLE] XNU hybrid kernel · [TI] [QCOM] boot flow.
> **Prerequisites:** [COA 03 ISA](../Computer%20Organization%20and%20Architecture/03-ISA-and-Instruction-Formats.md), [COA 13 Interrupts](../Computer%20Organization%20and%20Architecture/13-IO-Interrupts-DMA.md).

---

## 1. Intuition
The OS is like a **bank**. Customers (programs) can't walk into the vault (hardware) themselves. They go to the counter (a **system call**) and ask: "read this file", "give me memory", "send this packet". A teller (the **kernel**), who alone has the vault keys (**privileged mode**), checks the request and does it. The bank also has an alarm clock (the **timer interrupt**) so no customer can hog the counter forever.

The OS has three jobs:
1. **Abstraction:** turn messy hardware into clean objects: process, thread, address space, file, socket.
2. **Resource management:** share the CPU, memory, disk and network fairly and efficiently.
3. **Protection:** stop programs from harming each other or the kernel.

---

## 2. Deep Dive

### 2.1 Dual mode: user mode vs kernel mode
The CPU has a **mode bit** (privilege level). Some instructions only work in kernel mode.

| | User mode | Kernel mode |
|---|---|---|
| Who runs here | your applications, libc | the kernel, device drivers |
| Can execute privileged instructions? | ❌ (trap → kernel kills/handles) | ✅ |
| Examples of privileged ops | — | change page tables (CR3 / TTBR), disable interrupts (`cli`), talk to devices (I/O ports/MMIO), halt the CPU, change mode |
| Memory it can touch | only its own address space (user pages) | everything |

**Privilege levels per ISA:**
| ISA | User | Kernel | Hypervisor | Firmware/secure |
|---|---|---|---|---|
| x86 | ring 3 | ring 0 | VMX root (ring "−1") | SMM |
| ARMv8 | EL0 | EL1 | EL2 | EL3 (secure monitor, TrustZone) |
| RISC-V | U | S | (H extension) | M |

**Why dual mode?** Without it, any buggy or malicious program could overwrite the kernel, read other processes' memory, or disable interrupts and never give back the CPU.

### 2.2 Three ways to enter the kernel
| Event | Cause | Sync / async | Example | After handling |
|---|---|---|---|---|
| **System call (trap)** | program **asks** on purpose | synchronous | `read()`, `write()`, `fork()` | return to the next instruction |
| **Exception (fault)** | instruction **went wrong** | synchronous | page fault, divide by zero, illegal instruction, segfault | **re-execute** the instruction (page fault) or kill the process (SIGSEGV/SIGFPE) |
| **Interrupt** | **device / timer** needs attention | asynchronous (any time) | timer tick, NIC packet arrived, disk done | return to whatever was running (maybe after a context switch) |

> **"All three feel the same. What does sync/async mean?"** They use the **same mechanism** (save state → kernel mode → handler). The difference is **who caused it and when**, measured against the instruction being executed:
> - **Synchronous** = caused **by the current instruction**, happens **at that exact instruction, every run** (syscall, page fault, divide by zero).
> - **Asynchronous** = caused by something **outside** the program (timer, device, another core), arrives **between any two instructions**, at a different place each run.
> ```
> Program: I1 add   I2 load x (page absent)   I3 syscall   I4 div by 0
> RUN 1:  I1 → I2 ⚡fault → I3 ⚡syscall → I4 ⚡div      timer IRQ arrived between I1 and I2
> RUN 2:  I1 → I2 ⚡fault → I3 ⚡syscall → I4 ⚡div      timer IRQ arrived between I3 and I4
>         same place every run = SYNC                   different place each run = ASYNC
> ```
> Analogy: phoning a shop (syscall) or cutting your finger (exception) is caused by what you're doing now; the doorbell (interrupt) can ring at any moment.
> | | Synchronous (syscall, exception) | Asynchronous (interrupt) |
> |---|---|---|
> | Return to | fault ⇒ **same** instruction (retry); trap ⇒ **next** instruction | wherever the program was interrupted |
> | On behalf of | the **current** process (it gets SIGSEGV / made the call) | usually **not** the current process; the handler borrows the CPU (interrupt context) |
> | Can be postponed? | **no**: the instruction can't proceed without it | **yes**: interrupts can be **masked** (`cli`) |
> | Debugging | reproducible | timing-dependent (races with ISRs) |
> **Naming (Intel):** *interrupts* = async; *exceptions* = sync, split into **faults** (retry: page fault), **traps** (next instruction: `syscall`, `int3`), **aborts** (fatal: double fault, machine check). A syscall is a *trap*. Old `int 0x80` was called a "software interrupt", but it's still **synchronous**.
> Interrupts are only taken **between** instructions; long instructions like `rep movsb` can be interrupted between iterations and resumed.

**The timer interrupt is how the OS keeps control.** A user program in `while(1);` never makes a syscall. Every ~1–4 ms the timer interrupt fires, the kernel runs, and the scheduler can switch to another process. This is **preemption**.

> **Is the timer hardware or software? The interrupt source is HARDWARE; software timers are built on top of it.**
> **Why it must be hardware:** software runs only when the CPU executes it. During `while(1);` the kernel never runs, so no software could "fire". An external device has to count time on its own and **force** the CPU into the kernel with an interrupt.
> ```
>  crystal/oscillator ──► COUNTER ──► COMPARE with the value the kernel programmed ──► IRQ ──► CPU
>                                                                    ──► kernel timer interrupt handler
> ```
> The kernel programs it as **periodic** ("every 1 ms") or **one-shot** ("at time T"). Programming the timer and masking interrupts are **privileged**, so user code can't disable preemption.
> | Platform | Hardware timer |
> |---|---|
> | x86 old / modern | PIT 8254 (1.193182 MHz), HPET / **Local APIC timer per core** (TSC-deadline mode) |
> | ARMv8 | **Generic Timer** (per-core counter + compare) |
> | Cortex-M (TI/QCOM MCUs) | **SysTick** 24-bit down-counter (the FreeRTOS tick) |
> | RISC-V | `mtime`/`mtimecmp` (CLINT) or `stimecmp` (Sstc) |
> Linux names two roles: **clocksource** = a counter you *read* for time (TSC; no interrupts), **clockevent** = a device that *generates* interrupts (LAPIC timer). The **RTC** is a battery-backed clock for wall time while powered off.
>
> **Software timers** (`sleep`, socket timeouts, `alarm`, `timerfd`, TCP retransmits, RTOS software timers): thousands of them are **multiplexed on one hardware timer per core**. The kernel keeps them sorted (Linux: **timer wheel** for coarse `timer_list`, **RB-tree** for `hrtimers`) and programs the hardware for the **earliest** expiry. Each interrupt runs the expired timers, does the scheduler-tick work, and re-arms the hardware for the next event.
> - **Periodic tick:** fires every 1/HZ regardless. **Tickless (`NO_HZ`):** one-shot, programmed only for the next real event ⇒ power saving, less jitter (`nohz_full`, note 13).
> - So: **interrupt = hardware; policy (what to do on each tick, when to fire next) = software.**
>
> **SysTick numerical [TI][QCOM]:** 48 MHz CPU, 1 ms tick ⇒ 48,000 counts ⇒ RELOAD = **47,999** (counts RELOAD → 0 = RELOAD + 1 cycles). Max period = 2²⁴ / 48 MHz ≈ **0.35 s**. *Try:* 80 MHz, 500 µs ⇒ RELOAD = 39,999; max ≈ 0.21 s.

### 2.3 What happens on a system call, step by step (x86-64 Linux)
```c
write(1, "hi\n", 3);        // your code
```
```
USER MODE
 1. libc's write() wrapper puts:  rax = 1 (syscall number of write)
                                  rdi = 1 (fd), rsi = buf, rdx = 3 (count)
 2. executes the `syscall` instruction
──────────────────────── mode switch (hardware) ────────────────────────
KERNEL MODE
 3. CPU: switch to ring 0, save user RIP in rcx and RFLAGS in r11, jump to the
    kernel entry point (address stored in an MSR at boot)
 4. Entry code: switch to this thread's kernel stack, save user registers
 5. Look up sys_call_table[rax] → sys_write
 6. sys_write: validate fd and buffer pointer, copy_from_user(), hand bytes to
    the terminal/file/socket driver
 7. Put the return value in rax (bytes written, or −errno on error)
 8. Restore user registers, `sysret`
──────────────────────── mode switch back ─────────────────────────────
USER MODE
 9. libc wrapper: if rax is negative → set errno, return −1; else return rax
```
| ISA | Instruction | Syscall number in | Args in |
|---|---|---|---|
| x86-64 | `syscall` | `rax` | `rdi, rsi, rdx, r10, r8, r9` |
| ARM64 | `svc #0` | `x8` | `x0–x5` |
| RISC-V | `ecall` | `a7` | `a0–a5` |

**Why does the kernel copy user data (`copy_from_user`) instead of using the pointer directly?** The pointer comes from an untrusted program. It might point into kernel memory, be unmapped, or be changed by another thread mid-call. Copying with checks prevents the kernel from being tricked.

### 2.4 Library call vs system call
| | Library call (`printf`, `strlen`, `malloc`) | System call (`write`, `brk`, `mmap`) |
|---|---|---|
| Runs in | user mode | kernel mode |
| Cost | a normal function call (ns) | mode switch + kernel work (~100 ns–µs) |
| Example chain | `printf` → formats into a user buffer → when the buffer is full or has `\n` (terminal) → **`write()`** syscall | |
| Tool to trace | `ltrace` | `strace` |
`malloc` usually does **not** make a syscall. It hands out memory from a heap it already owns, and only calls `brk`/`mmap` when it needs more from the kernel (note 08).

### 2.5 What does a syscall cost? [HFT]
| Item | Typical cost |
|---|---|
| Function call | ~1 ns |
| Bare syscall (e.g. `getpid`) on modern x86 | **~50–100 ns**; a few hundred ns with Meltdown/Spectre mitigations (KPTI) |
| `read()` of cached file data | ~0.5–2 µs |
| Hidden cost | the kernel pollutes L1/L2 caches, the TLB and branch predictors, so your code runs slower **after** the syscall |
- **vDSO** (virtual dynamic shared object): the kernel maps a small page of code and data into every process so that `clock_gettime()`, `gettimeofday()` and `getcpu()` run **in user mode with no mode switch** (~20 ns). HFT timestamps use this, or `rdtsc` directly.
- **HFT rule:** no syscalls on the hot path. Pre-allocate memory, busy-poll instead of blocking, use kernel-bypass networking (note 11, note 13).
- **Batching:** one `write()` of 4 KB instead of 4096 `write()`s of 1 byte (that's why stdio buffers).

### 2.6 Kernel architectures
```
 MONOLITHIC (Linux)            MICROKERNEL (QNX, seL4, MINIX)       HYBRID (Windows NT, Apple XNU)
┌───────────────────────┐     ┌─────┐┌─────┐┌──────┐┌──────┐        Mach microkernel core +
│ apps (user mode)      │     │ app ││ FS  ││driver││ net  │ user   BSD layer + IOKit drivers
├───────────────────────┤     └──┬──┘└──┬──┘└──┬───┘└──┬───┘        all in kernel space
│ FS, net, drivers, MM, │        └──────┴─IPC─┴───────┘             (microkernel ideas, monolithic
│ scheduler: ALL in     │     ┌────────────────────────┐             speed)
│ kernel space          │     │ tiny kernel: IPC,      │ kernel
└───────────────────────┘     │ scheduling, basic MM   │
                              └────────────────────────┘
```
| | Monolithic | Microkernel | Hybrid |
|---|---|---|---|
| Speed | fast (a function call between subsystems) | slower (IPC message for each service request) | close to monolithic |
| Reliability | a driver bug can crash the whole kernel | a driver crash = restart that server | in between |
| Examples | Linux, BSDs | QNX (cars), seL4 (formally verified), MINIX (inside Intel ME) | Windows NT, **macOS/iOS XNU** [APPLE] |
- **Loadable kernel modules** (Linux `.ko`, `insmod`): drivers added at run time, still running in kernel space. That's monolithic, but modular.

### 2.7 Kernel and user address space
On 64-bit Linux, every process's virtual address space has user memory in the lower half and the **kernel mapped into the upper half** (same in every process, accessible only in kernel mode).
- So a syscall doesn't need to switch page tables: the kernel is "already there".
- **Meltdown (2018)** let user code speculatively read those kernel mappings, so **KPTI** keeps the kernel mostly unmapped while in user mode. That makes syscalls more expensive (a page table switch on every entry/exit; PCID reduces the TLB flush cost). See [COA 09 Spectre/Meltdown](../Computer%20Organization%20and%20Architecture/09-ILP-Superscalar-Out-of-Order.md).

### 2.8 Boot process [TI][QCOM][APPLE]
```
PC / server:  power on → CPU starts at reset vector in firmware (UEFI/BIOS)
              → POST, init RAM → load bootloader (GRUB) → load kernel + initramfs
              → kernel: init MMU, interrupts, drivers, mount root FS
              → start PID 1 (systemd/init) → services → login
Embedded Linux (QCOM SoC, TI Sitara):
              Boot ROM (on-chip, immutable) → SPL / first-stage loader (init DRAM)
              → U-Boot → Linux kernel + device tree (.dtb) → init
Microcontroller (TI MSP430 / Cortex-M, bare metal or RTOS):
              reset → read vector table (initial SP, reset handler) → startup code
              (copy .data from flash to RAM, zero .bss, set clocks) → main() / RTOS scheduler start
Secure boot: each stage verifies the signature of the next (chain of trust from Boot ROM keys).
```

### 2.9 Common syscalls to know by name
| Area | Syscalls |
|---|---|
| Processes | `fork`, `execve`, `wait4`/`waitpid`, `exit`, `clone`, `kill`, `getpid` |
| Files | `open`, `read`, `write`, `close`, `lseek`, `stat`, `fsync`, `dup2`, `ioctl` |
| Memory | `brk`, `mmap`, `munmap`, `mprotect`, `madvise`, `mlock` |
| I/O multiplexing | `select`, `poll`, `epoll_create/ctl/wait`, `io_uring_enter` |
| IPC / net | `pipe`, `socket`, `bind`, `connect`, `sendmsg`, `shm_open` (via `open` + `mmap`), `futex` |

---

## 3. Worked Numericals

**N1. Syscall overhead of unbuffered output.** A program writes 10⁶ log lines of 50 B each. (a) one `write()` per line, syscall cost 1 µs; (b) buffered into 64 KB chunks.
- (a) 10⁶ × 1 µs = **1 s** of pure syscall overhead.
- (b) Total = 50 × 10⁶ B = 50 MB; chunks = 50 MB / 64 KB ≈ 763 → 763 × 1 µs ≈ **0.76 ms**.
- Speed-up of overhead ≈ **1300×**. This is why stdio buffers, and why HFT loggers write to a shared-memory ring buffer and let another thread do the `write()`.

**N2. Timer interrupt overhead.** HZ = 1000 (tick every 1 ms), each tick handler takes 2 µs.
Overhead = 2 µs / 1 ms = **0.2 %** of each CPU. Small for throughput, but for HFT that's a **2 µs jitter** spike every millisecond on the trading core ⇒ use `nohz_full` (no tick on isolated cores, note 13).

**N3. Mode switches when copying a file.** `cp` copies a 1 MB file with a 4 KB buffer using `read` + `write`.
Number of `read` calls = 1 MB / 4 KB = 256 (plus 1 that returns 0 = EOF) → 257. `write` calls = 256. Total ≈ **513 syscalls ⇒ 1026 mode switches** (each syscall = enter + exit). With a 128 KB buffer: 8 + 1 + 8 = 17 syscalls. (`sendfile`/`copy_file_range` does it in ~1, note 11.)

**Try it:** a market-data handler calls `clock_gettime()` 2 million times per second. Real syscall at 150 ns vs vDSO at 20 ns: what fraction of one core does each use?
<details><summary>Answer</summary>Syscall: 2×10⁶ × 150 ns = 0.3 s per second = <b>30 %</b> of a core. vDSO: 2×10⁶ × 20 ns = 0.04 s = <b>4 %</b>.</details>

---

## 4. Interview Questions

### Basic
- **Q: What is a system call?** A controlled entry into the kernel. A user program executes a trap instruction (`syscall`/`svc`/`ecall`) with a syscall number and arguments. The CPU switches to kernel mode at a fixed entry point, and the kernel validates and performs the request.
- **Q: Why do we need user and kernel modes?** Protection. Only the kernel can run privileged instructions (page tables, I/O, interrupts), so buggy or malicious programs can't take over the machine.
- **Q: Trap vs interrupt vs exception?** Trap = an intentional synchronous syscall. Exception = a synchronous error caused by an instruction (page fault, divide by zero). Interrupt = an asynchronous event from a device or timer. Follow-up: *"What does synchronous mean here?"* Caused by the current instruction, so it happens at the same place every run; async arrives between any two instructions and can be masked (§2.2 box).
- **Q: Is `printf` a system call?** No. It's a library function that formats into a user-space buffer, then calls the `write()` syscall when the buffer is flushed.

### Intermediate
- **Q: How does the OS regain control from a process stuck in `while(1)`?** The timer interrupt. The hardware forces entry to the kernel periodically, and the scheduler can preempt the process.
- **Q: Is the timer that preempts processes hardware or software?** Hardware (LAPIC timer / ARM Generic Timer / SysTick) raises the interrupt; software decides the policy. Follow-up: *"Then how can 10,000 threads sleep with different deadlines?"* Software timers kept in a timer wheel / RB-tree; the hardware is programmed for the earliest expiry and re-armed after each interrupt (§2.2 box).
- **Q: Monolithic vs microkernel?** Table §2.6. Linux is monolithic with loadable modules; QNX/seL4 are microkernels; XNU and NT are hybrids.
- **Q [APPLE]: What is XNU?** The macOS/iOS kernel: the Mach microkernel (tasks, threads, IPC via Mach ports, VM) + a BSD layer (POSIX, processes, networking, VFS) + IOKit (a C++ driver framework), all running in kernel space ⇒ a hybrid.
- **Q [HFT]: How expensive is a syscall and how do you avoid it?** ~100 ns to µs, plus cache/TLB pollution. Avoid: vDSO for time, buffering/batching, memory pre-allocation, busy-polling, kernel-bypass networking (DPDK/OpenOnload), shared-memory IPC.

### Advanced / killer follow-ups
- **Q: Walk me through `write(fd, buf, n)` from user space to the device.** §2.3 steps → VFS → filesystem → page cache (returns as soon as the data is in the page cache!) → writeback later → block layer → driver → DMA to disk. Follow-up: *"So when is the data actually on disk?"* Only after `fsync()` (note 10).
- **Q: Why can't the kernel just dereference a user pointer?** It's untrusted: it could point into kernel memory, be unmapped, or change concurrently. Use `copy_from_user`, which checks the range and handles faults safely.
- **Q: What is the vDSO and why does it exist?** §2.5: kernel-provided user-mode code for read-only, frequently called syscalls like the time functions; no mode switch.
- **Q [NVIDIA][QCOM]: What can't you do in kernel code that you can in user code?** No libc, small fixed kernel stack (8–16 KB), no floating point by default, can't sleep in interrupt context, a bug crashes the whole machine (note 13).
- **Q: What did KPTI change and why?** It unmaps the kernel while user code runs (Meltdown defence), so every syscall/interrupt switches page tables ⇒ higher syscall cost.

---

## 5. Common Traps
- Calling `malloc` or `printf` a system call. They're library calls that sometimes make syscalls.
- Thinking `write()` returning means the data is on disk (it's only in the page cache).
- Forgetting the timer interrupt when asked how preemption works.
- Saying a page fault always means a crash. Most page faults are normal (demand paging, CoW) and invisible (note 09).
- Mixing up mode switch (user↔kernel, same process) with context switch (process A → process B). A syscall is a mode switch, not necessarily a context switch.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| OS jobs | abstraction, resource management, protection |
| Dual mode | user (ring 3 / EL0) vs kernel (ring 0 / EL1); privileged instructions trap |
| Kernel entry | syscall (intentional) · exception (fault) · interrupt (async device/timer) |
| Sync vs async | sync = caused by the current instruction, same place every run, can't be postponed; async = external, between any two instructions, maskable |
| Preemption | the timer interrupt gives the kernel control back |
| Timer | hardware raises the IRQ (LAPIC/Generic Timer/SysTick); software timers multiplexed on it (wheel/RB-tree); periodic vs tickless |
| Syscall path | number in `rax`, args in regs → `syscall` → table → handler → `sysret`; error = −errno |
| Cost | ~100 ns bare, µs with work + cache pollution; vDSO avoids it for time |
| Lib vs syscall | `printf`/`malloc` = library, call `write`/`brk`/`mmap` sometimes |
| Kernels | monolithic (Linux), micro (QNX, seL4), hybrid (NT, XNU) |
| Boot | ROM → firmware/SPL → bootloader (GRUB/U-Boot) → kernel → PID 1 |
| Mode vs context switch | syscall = mode switch; switching processes = context switch |
