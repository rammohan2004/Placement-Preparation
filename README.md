# Placement Preparation — Master Roadmap

**Target companies:** NVIDIA · Texas Instruments · Qualcomm · Amazon · Apple · HFTs (IMC Trading, Optiver, Tower, Graviton, Quadeye) · D. E. Shaw

## Subject order (and why)
| # | Subject | Why this order | Heaviest at |
|---|---|---|---|
| 1 | **Computer Organization & Architecture** | Foundation for OS, low-latency C++, GPU work | NVIDIA, Apple, Qualcomm, TI, HFT |
| 2 | Operating Systems | Builds on VM, caches, interrupts from COA | All |
| 3 | Digital Logic Design (+ Verilog basics) | Must for HW roles | NVIDIA HW, TI, Qualcomm, Apple HW |
| 4 | C and C++ for Systems | Memory model, UB, templates, low latency | HFT, D. E. Shaw, NVIDIA SW, Qualcomm |
| 5 | Computer Networks | Concept rounds; TCP/UDP for HFT | Amazon, D. E. Shaw, HFT |
| 6 | DBMS | Concept rounds | Amazon, D. E. Shaw |
| ∥ | DSA (daily, in parallel) | Every OA and coding round | All |
| ∥ | Probability, puzzles, mental math (in parallel) | Quant / HFT rounds | IMC, Optiver, D. E. Shaw |

## Progress
Status: ⬜ not started · 🟨 in progress · ✅ done. Confidence 1–5.

### Computer Organization & Architecture → [index](<CoreCS/Computer Organization and Architecture/00-README.md>)
| # | Topic | Status | Confidence | Date |
|---|---|---|---|---|
| 01 | Number Representation & Arithmetic | ✅ | | 2026-11-09 |
| 02 | Performance Metrics & Laws | ✅ | | 2026-11-09 |
| 03 | ISA & Instruction Formats | ✅ | | 2026-11-09 |
| 04 | Pipelining & Hazards | ✅ | | 2026-11-09 |
| 05 | Branch Prediction | ✅ | | 2026-11-09 |
| 06 | Cache Memory | ✅ | | 2026-11-09 |
| 07 | Virtual Memory & TLB | ✅ | | 2026-11-09 |
| 08 | Main Memory (DRAM, HBM) | ✅ | | 2026-11-09 |
| 09 | ILP, Superscalar & Out-of-Order | ✅ | | 2026-11-09 |
| 10 | Cache Coherence & Memory Consistency | ✅ | | 2026-11-09 |
| 11 | SIMD, Multithreading & Multicore | ✅ | | 2026-11-09 |
| 12 | GPU Architecture | ✅ | | 2026-11-09 |
| 13 | I/O, Interrupts & DMA | ✅ | | 2026-11-09 |
| 14 | Low-Latency Hardware Awareness (HFT) | ✅ | | 2026-11-09 |
| 15 | Interview Question Bank | ✅ | | 2026-11-09 |
| 16 | Cheat Sheet | ✅ | | 2026-11-09 |

### Operating Systems → [index](<CoreCS/Operating Systems/00-README.md>)
| # | Topic | Status | Confidence | Date |
|---|---|---|---|---|
| 01 | OS Basics, Kernel & System Calls | ⬜ | | |
| 02 | Processes | ⬜ | | |
| 03 | Threads | ⬜ | | |
| 04 | CPU Scheduling | ⬜ | | |
| 05 | Synchronization Primitives | ⬜ | | |
| 06 | Classic Synchronization Problems | ⬜ | | |
| 07 | Deadlocks | ⬜ | | |
| 08 | Memory Management | ⬜ | | |
| 09 | Virtual Memory: Page Faults & Replacement | ⬜ | | |
| 10 | File Systems | ⬜ | | |
| 11 | I/O Systems, I/O Models & Disk Scheduling | ⬜ | | |
| 12 | IPC & Signals | ⬜ | | |
| 13 | Linux Internals & Low-Latency Tuning | ⬜ | | |
| 14 | RTOS & Embedded OS | ⬜ | | |
| 15 | Virtualization & Containers | ⬜ | | |
| 16 | Interview Question Bank | ⬜ | | |
| 17 | Cheat Sheet | ⬜ | | |

## Skills available in this workspace (`.claude/skills/`)
Ask Claude in plain words; these trigger automatically (or type `/skill-name`).
| Skill | Use it like |
|---|---|
| `study-notes` | "Write notes on TLB shootdown and add to the VM note" |
| `concept-deep-dive` | "Why exactly do we need a reorder buffer?" |
| `numerical-practice` | "Give me 5 cache numericals" / "Check my AMAT solution" |
| `mock-interview` | "Mock interview me on caches, NVIDIA style" |
| `revision-flashcards` | "Flashcards for pipelining" / "Rapid fire on coherence" |
| `company-prep` | "I have a Qualcomm interview in 5 days — plan it" |
| `progress-tracker` | "Mark caches done, confidence 4. What next?" |
