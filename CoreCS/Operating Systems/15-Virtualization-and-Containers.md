# 15. Virtualization & Containers

> **Asked at:** [AMZN] VM vs container, hypervisor types, EC2/Nitro, Firecracker (Lambda), cgroups/namespaces, noisy neighbours · [NVIDIA] GPU virtualization (vGPU, MIG), containers for ML (NVIDIA Container Toolkit), SR-IOV · [QCOM] [APPLE] hypervisors on phones (Android pKVM, Apple Virtualization framework) · [HFT] why trading runs on bare metal.
> **Prerequisites:** [01 OS Basics](01-OS-Basics-Kernel-and-System-Calls.md) (privilege levels), [08](08-Memory-Management.md)/[09](09-Virtual-Memory-Page-Faults-and-Replacement.md) memory, [COA 07 page walks](../Computer%20Organization%20and%20Architecture/07-Virtual-Memory-and-TLB.md).

---

## 1. Intuition
- A **virtual machine** is a **separate house built inside a big building**: it has its own kitchen, plumbing and front door (its own OS kernel). The building manager (the **hypervisor**) splits the real utilities between houses. Strong isolation, but each house carries a full kernel.
- A **container** is a **separate apartment in one house**: its own room, lock and name on the door, but everyone shares the house's plumbing (the **host kernel**). Lightweight and fast to start, but weaker walls.

---

## 2. Deep Dive

### 2.1 Hypervisor types
| | Type 1 (bare metal) | Type 2 (hosted) |
|---|---|---|
| Runs on | hardware directly | a host OS, as an application |
| Examples | VMware ESXi, Xen, Microsoft Hyper-V, **KVM** (Linux kernel module turning Linux into a type-1-style hypervisor), AWS **Nitro** hypervisor | VirtualBox, VMware Workstation/Fusion, Parallels, QEMU (without KVM) |
| Use | data centres / clouds | desktops, development |
| Overhead | lower | higher |

### 2.2 How do you run a kernel inside a VM? (CPU virtualization)
The guest kernel thinks it's in ring 0 and runs privileged instructions (load page-table base, disable interrupts, I/O). It must **not** actually control the real hardware.
| Technique | Idea | Notes |
|---|---|---|
| **Trap-and-emulate** | run the guest kernel deprivileged; each privileged instruction **traps** to the hypervisor, which emulates it | works only if every *sensitive* instruction traps (Popek–Goldberg, 1974). Classic x86 failed this (e.g. `popf` silently ignored the interrupt flag in user mode) |
| **Binary translation** | the hypervisor rewrites problematic guest kernel code on the fly | early VMware |
| **Paravirtualization** | modify the guest OS to call the hypervisor explicitly (**hypercalls**) instead of using privileged instructions | Xen; today's **virtio** drivers are paravirtual I/O |
| **Hardware-assisted** (Intel **VT-x**, AMD-V, ARM EL2) | a new CPU mode: **VMX root** (hypervisor) vs **non-root** (guest, with its own rings 0–3). Sensitive operations cause a **VM exit** to the hypervisor; it handles them and does **VM entry** | universal today; a VM exit costs ~1 µs (hundreds to thousands of cycles), so hypervisors minimise exits |

### 2.3 Memory virtualization
Three address spaces: **guest virtual (GVA) → guest physical (GPA) → host physical (HPA)**.
| | Shadow page tables (old) | Nested paging: Intel **EPT** / AMD **NPT** / ARM stage-2 |
|---|---|---|
| How | the hypervisor builds a combined GVA → HPA table and keeps it in sync by trapping guest page-table writes | hardware walks **two** tables: the guest's (GVA → GPA) and the hypervisor's (GPA → HPA) |
| Cost | many VM exits on page-table updates | no exits for guest page-table changes, but **a TLB miss is more expensive** |
**2-D page walk:** each guest page-table level address is itself a GPA that needs a host walk. With 4-level guest and 4-level host tables, a worst-case TLB miss takes **(4+1)(4+1) − 1 = 24 memory references** (vs 4 natively) ⇒ large pages (on the host side especially) matter a lot in VMs.
- **Memory overcommit tricks:** ballooning (a guest driver "inflates" to give pages back), page sharing/KSM (deduplicate identical pages), swapping.

### 2.4 I/O virtualization
| Technique | How | Performance |
|---|---|---|
| Full emulation | the hypervisor emulates a real device (e.g. an e1000 NIC) in software; every register access is a VM exit | slow |
| **Paravirtual (virtio)** | the guest uses a virtio driver; shared-memory ring queues + notifications | good |
| **Device passthrough** | give the VM a whole physical device; the **IOMMU** (VT-d) translates the device's DMA addresses safely | near native |
| **SR-IOV** | one physical device presents many **virtual functions** (VFs), each passed through to a VM | near native, shareable |
**AWS Nitro** [AMZN]: moves networking, storage (EBS) and management off the host CPU onto dedicated **Nitro cards** (SR-IOV ENA NICs, NVMe), plus a minimal KVM-based hypervisor ⇒ nearly all host CPU goes to customers, with near-bare-metal performance; bare-metal instances also exist.
**[NVIDIA] GPU sharing:** time-sliced vGPU, **MIG** (Multi-Instance GPU on A100/H100: hardware partitions with their own SMs, memory and cache slices ⇒ isolation + QoS), passthrough.

### 2.5 Containers: namespaces + cgroups + layered filesystem
A container is just **ordinary Linux processes** with:
**1. Namespaces: what a process can SEE** (isolation of views)
| Namespace | Isolates |
|---|---|
| `pid` | process IDs (the container's first process is PID 1 inside) |
| `net` | network interfaces, IPs, routing, ports |
| `mnt` | mount points / filesystem view |
| `uts` | hostname |
| `ipc` | System V IPC, POSIX message queues |
| `user` | UID/GID mapping (root inside ≠ root outside) |
| `cgroup` | view of the cgroup hierarchy |
| `time` | clock offsets |

**2. cgroups (control groups): what a process can USE** (resource limits and accounting)
- CPU: `cpu.max = "50000 100000"` ⇒ 50 ms per 100 ms period = **0.5 CPU**; `cpu.weight` for proportional shares; `cpuset` to pin to cores.
- Memory: `memory.max` hard limit ⇒ exceeding it triggers the **OOM killer inside the cgroup**; `memory.high` throttles.
- I/O bandwidth/IOPS (`io.max`), number of processes (`pids.max`).

**3. Layered image filesystem:** **overlayfs** stacks read-only image layers + one writable layer per container (copy-on-write) ⇒ many containers share the same base image on disk and in the page cache.

Plus security hardening: **seccomp** (syscall filtering), capabilities (drop root powers), AppArmor/SELinux.
Tooling: Docker/containerd/runc create these; **Kubernetes** schedules containers (pods) across machines, using cgroups for requests/limits.

### 2.6 VM vs container
| | Virtual machine | Container |
|---|---|---|
| Kernel | own guest kernel | **shares the host kernel** |
| Isolation | strong (hardware-enforced boundary) | weaker (a kernel bug can break out) |
| Start-up | seconds–minutes (boot an OS) | milliseconds (start a process) |
| Size | GBs (full OS image) | MBs (app + libraries) |
| Overhead | VM exits, 2-D page walks, virtual devices | near native (just processes) |
| Run a different OS? | yes (Windows on Linux) | no (Linux containers need a Linux kernel) |
| Density | tens per host | hundreds–thousands per host |

**Best of both: microVMs.** **AWS Firecracker** (Rust, on KVM) [AMZN] boots a minimal VM in ~125 ms with ~5 MB memory overhead and a tiny device model (virtio net/block only) ⇒ VM-level isolation at near-container speed. It powers **AWS Lambda** and Fargate, where code from different customers must be strongly isolated. Similar: gVisor (a user-space kernel that intercepts syscalls), Kata Containers.

### 2.7 Why HFT runs on bare metal [HFT]
VM exits, 2-D page walks, virtual interrupt delivery, noisy neighbours on shared caches and memory bandwidth, and the hypervisor's own scheduling ⇒ extra µs and jitter. Trading systems use dedicated bare-metal servers in the exchange's colocation facility, often with kernel bypass (note 11/13). Containers are sometimes used for deployment, but with pinned cores, host networking and no CPU limits (CFS quota throttling causes latency spikes).

---

## 3. Worked Numericals

**N1. 2-D page walk cost.** Guest and host both use 4-level tables, no paging-structure caches.
Native TLB miss = 4 memory references. Nested = (4 + 1)(4 + 1) − 1 = **24**. At 80 ns each (worst case, all DRAM): 320 ns vs 1.92 µs. With 2 MB host pages (3-level host walk): (4 + 1)(3 + 1) − 1 = **19**.

**N2. cgroup CPU quota throttling.** A container has `cpu.max = 200000 100000` (2 CPUs' worth per 100 ms). A multithreaded service runs 8 threads at once for a burst.
8 threads consume the 200 ms quota in 200/8 = **25 ms** of wall time ⇒ throttled for the remaining **75 ms** of the period ⇒ a request arriving then waits up to 75 ms ⇒ p99 latency spikes. Fixes: a larger quota, fewer threads, or `cpuset` pinning instead of a quota.

**N3. Density.** A 64 GB host. VMs need 2 GB guest OS + 1 GB app = 3 GB ⇒ ~21 VMs. Containers sharing the kernel and base-image page cache need ~1.1 GB each ⇒ ~58 containers (≈2.8× density).

**N4. VM exits.** A VM does 200,000 virtio notifications/s, each causing a VM exit of 1.5 µs ⇒ 0.3 s/s = **30 %** of a core lost ⇒ batch notifications (event index / interrupt suppression), use vhost (in-kernel backend), or SR-IOV passthrough (no exits on the data path).

**Try it:** a pod has `cpu.max = 50000 100000` and its single-threaded request handler needs 80 ms of CPU per request. Minimum wall-clock latency of one request?
<details><summary>Answer</summary>50 ms per 100 ms period. It runs 50 ms, throttled 50 ms, then runs the remaining 30 ms ⇒ ≈ <b>130 ms</b> (vs 80 ms unthrottled).</details>

---

## 4. Interview Questions

### Basic
- **Q: VM vs container?** §2.6 table: own kernel vs shared kernel.
- **Q: Type 1 vs Type 2 hypervisor?** §2.1.
- **Q: What are namespaces and cgroups?** Namespaces limit what you can see; cgroups limit what you can use.

### Intermediate
- **Q: How does hardware-assisted virtualization work?** VT-x/AMD-V root/non-root modes, VM exits on sensitive operations, VM entry back.
- **Q: What is EPT/NPT and why does it help? What does it cost?** Hardware two-level translation, no shadow page tables or exits on guest page-table writes; costs a longer TLB-miss walk (24 references) ⇒ use huge pages.
- **Q: Paravirtualization / virtio?** The guest knows it's virtualized and uses efficient shared-ring interfaces instead of emulated hardware.
- **Q: What happens when a container exceeds its memory limit?** The cgroup OOM killer kills a process in that container (exit code 137 in Docker/Kubernetes: `OOMKilled`).

### Advanced / killer follow-ups
- **Q [AMZN]: Why does AWS Lambda use Firecracker microVMs instead of plain containers?** Multi-tenant untrusted code needs VM-level (hardware) isolation; Firecracker gives that with ~125 ms boot and a few MB of overhead.
- **Q [AMZN]: What is Nitro?** Offloads networking/storage/management to dedicated cards + a lightweight KVM hypervisor ⇒ near bare-metal performance and stronger security.
- **Q: Can a container run a Windows app on a Linux host?** Not natively (shared Linux kernel); you need a VM.
- **Q: Container security: why is "root in a container" risky?** Same kernel; without user namespaces, root inside = root outside for kernel purposes; kernel exploits break out ⇒ seccomp, drop capabilities, user namespaces, gVisor/Kata/Firecracker for stronger isolation.
- **Q: Latency spikes in a Kubernetes service with CPU limits. Why?** CFS quota throttling (N2) ⇒ remove/raise limits, use static CPU manager (pinned cores).
- **Q [NVIDIA]: How do you share one GPU among several users/containers?** Time-slicing, vGPU, MIG partitions (hardware isolation), MPS (Multi-Process Service for concurrent kernels); containers access GPUs through the NVIDIA Container Toolkit (device files + driver libraries mounted in).

---

## 5. Common Traps
- "Containers are lightweight VMs". They're processes with namespaces + cgroups, sharing the host kernel.
- Thinking EPT removes all overhead (TLB misses get more expensive).
- Forgetting the IOMMU is required for safe device passthrough.
- Setting tight CPU limits on latency-sensitive containers (throttling).
- Saying Docker "virtualizes the CPU".

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Hypervisors | Type 1 bare metal (ESXi, Xen, KVM, Hyper-V, Nitro) vs Type 2 hosted (VirtualBox) |
| CPU virt | trap-and-emulate (Popek–Goldberg), binary translation, paravirt (hypercalls), VT-x/AMD-V (VM exit ~1 µs) |
| Memory virt | shadow tables → EPT/NPT; 2-D walk 24 refs; huge pages |
| I/O virt | emulation → virtio → passthrough (IOMMU) → SR-IOV |
| Nitro | offload cards + thin KVM hypervisor |
| Containers | namespaces (see), cgroups (use), overlayfs (layers), seccomp/capabilities |
| cgroup CPU | `cpu.max quota period`; throttling ⇒ latency spikes |
| VM vs container | own kernel/strong isolation/slow start vs shared kernel/weak/ms start |
| microVM | Firecracker: ~125 ms boot, Lambda/Fargate |
| GPU | vGPU, MIG, MPS, passthrough |
| HFT | bare metal, colocation, no virtualization overhead |
