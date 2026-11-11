# 17. OS Cheat Sheet (one-screen revision)

## Formulas
| Topic | Formula |
|---|---|
| Turnaround time | TAT = completion − arrival |
| Waiting time | WT = TAT − burst |
| Response time | RT = first run − arrival |
| SJF prediction | τₙ₊₁ = α·tₙ + (1 − α)·τₙ |
| RR overhead | switch / (q + switch) |
| Max wait to run (RR) | (n − 1)·q |
| Thread pool size | cores × (1 + wait/compute) |
| Amdahl | S = 1 / (s + (1 − s)/N); max 1/s |
| EAT (paging) | (1 − p)·mem + p·fault_time |
| Page table size | 2^(VA bits − offset bits) × PTE size |
| Pages for a process | ⌈size / page size⌉; internal frag = pages·size − need |
| Proportional frames | aᵢ = sᵢ / S × m |
| Deadlock-free resources | R ≥ Σ(kᵢ − 1) + 1 (equal k: n(k − 1) + 1) |
| Banker's | Need = Max − Alloc; Available = Total − ΣAlloc; Work += Alloc |
| Inode max file | (direct + P + P² + P³) × block, P = block / ptr size |
| Disk access | seek + ½ rotation (= 30/RPM s) + transfer |
| RMS bound | U ≤ n(2^{1/n} − 1): 1, 0.828, 0.780, 0.757 … → 0.693 |
| Response-time analysis | Rᵢ = Cᵢ + Σ_{hp} ⌈Rᵢ/Tⱼ⌉·Cⱼ, iterate; need Rᵢ ≤ Dᵢ |
| EDF | schedulable ⇔ U ≤ 1 (D = T) |
| Fork count | n sequential forks ⇒ 2ⁿ processes |
| 2-D page walk | (n + 1)(m + 1) − 1 = 24 for 4 + 4 levels |

## Numbers to quote (typical, modern x86 Linux)
| Operation | Cost |
|---|---|
| L1 hit / DRAM access | ~1 ns / ~80–100 ns |
| Uncontended atomic / mutex lock+unlock | ~5–10 ns / ~20–50 ns |
| Cache-line transfer between cores | ~50–100 ns |
| Bare syscall | ~50–100 ns (more with KPTI) |
| vDSO `clock_gettime` | ~20 ns |
| Context switch (direct) | ~1–3 µs (+ cache/TLB warm-up) |
| Thread create / fork | ~10–20 µs / ~50–100+ µs |
| Minor page fault | ~0.5–2 µs |
| Major page fault (NVMe / HDD) | ~50–150 µs / ~5–10 ms |
| VM exit | ~1 µs |
| Interrupt entry/exit | ~1–5 µs total path |
| Kernel network stack round trip | ~5–20 µs; kernel bypass ~1–2 µs |
| Pipe / Unix socket message | ~2–5 µs; shared memory ring ~0.1 µs |
| HDD random 4 KB | ~10 ms (~100 IOPS); NVMe ~50–100 µs |
| Timer tick | 1–4 ms (HZ 1000–250) |
| Pipe buffer / PIPE_BUF | 64 KB / 4 KB |
| Stacks | user main 8 MB; kernel 16 KB; RTOS task: KBs |
| glibc mmap threshold | 128 KB |

## Classic numerical answers
| Problem | Answer |
|---|---|
| P1(0,8) P2(1,4) P3(2,9) P4(3,5): avg WT | FCFS 8.75 · SJF 7.75 · SRTF 6.5 · RR(q=4) 11.75 |
| 1 2 3 4 1 2 5 1 2 3 4 5, 3 frames | FIFO 9 · LRU 10 · OPT 7; FIFO 4 frames = 10 (Belady) |
| 7 0 1 2 0 3 0 4 2 3 0 3 2 1 2 0 1 7 0 1, 3 frames | FIFO 15 · LRU 12 · OPT 9 |
| Disk: head 53, 98 183 37 122 14 124 65 67 | FCFS 640 · SSTF 236 · SCAN (→0) 236 |
| Banker's (10,5,7) classic | safe: T1, T3, T4, T0, T2 |
| Holes 100 500 200 300 600; req 212 417 112 426 | only best fit places all |
| `fork() && fork() \|\| fork()` | 5 processes |
| `for(3) {fork(); printf("x\n");}` | 14 (terminal) / 24 (redirected to a file) |
| 4 KB blocks, 4 B pointers, 12 + 1 + 1 + 1 | ≈ 4 TB max file |

## One-liners
- **Process** = address space + resources + ≥1 thread; **thread** = registers + PC + stack + TLS.
- **Mode switch** (syscall) ≠ **context switch** (different task).
- **Timer interrupt** = how the OS preempts.
- **fork** returns 0 / child PID / −1; CoW; shares fd offsets. **exec** replaces the program, keeps PID.
- **Zombie**: exited, not reaped. **Orphan**: parent gone → init adopts.
- **Mutex** has an owner (+ priority inheritance); **semaphore** is a counter, anyone can signal.
- **Spin** if the expected wait < ~2 context switches; never spin on a single core.
- **Condvar**: wait = unlock + sleep atomically; always a `while` loop.
- **Deadlock**: mutual exclusion + hold & wait + no preemption + circular wait. Real fix: **lock ordering**.
- **Unsafe ≠ deadlock**. **RAG cycle = deadlock** only for single-instance resources.
- **Page fault ≠ segfault**; the faulting instruction is restarted.
- **LRU/OPT** = stack algorithms (no Belady); OSes use **Clock** approximations.
- **write() ≠ durable**: fsync; tmp + fsync + rename + fsync(dir).
- **Inode has no name**; space is freed when link count = 0 **and** no open fds.
- **epoll** = readiness, O(ready); **io_uring** = completion; **sendfile** = zero-copy.
- **Signal handlers**: only async-signal-safe calls; use a flag / self-pipe / signalfd.
- **Shared memory** = fastest IPC; store offsets, not pointers.
- **HFT recipe**: isolcpus + nohz_full + rcu_nocbs + IRQ affinity + pinning + mlockall/pre-fault + huge pages + C-states off + kernel bypass + measure p99.9 with rdtscp.
- **RTOS**: fixed-priority preemptive; ISR short, FromISR APIs; priority inheritance (Mars Pathfinder).
- **Containers** = namespaces (see) + cgroups (use) + overlayfs, shared kernel. **VM** = own kernel. **Firecracker** = microVM.
