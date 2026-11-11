# 02. Processes

> **Asked at:** [AMZN] [DESHAW] process vs program, states, fork/exec, zombie/orphan, "count the prints" fork puzzles · [HFT] context-switch cost · [NVIDIA] [QCOM] process address space, PCB · [APPLE] process lifecycle.
> **Prerequisites:** [01 OS Basics](01-OS-Basics-Kernel-and-System-Calls.md), [COA 07 VM](../Computer%20Organization%20and%20Architecture/07-Virtual-Memory-and-TLB.md) (copy-on-write §2.10).

---

## 1. Intuition
A **program** is a recipe written in a book (a file on disk: `/usr/bin/python`). A **process** is someone actually **cooking** that recipe: they have a kitchen (address space), ingredients in hand (registers, open files), and a bookmark showing where they are (program counter). Ten people can cook the same recipe at once = **ten processes of one program**. The OS is the kitchen manager who keeps a card for each cook (the **PCB**) and decides who gets the stove (CPU) next.

---

## 2. Deep Dive

### 2.1 Process = program in execution + its state
| A process owns | Details |
|---|---|
| **Address space** | its own virtual memory (code, data, heap, stack): private, isolated via page tables |
| **CPU context** | PC, stack pointer, general registers, flags (saved when not running) |
| **OS resources** | open file descriptors, sockets, signal handlers, current directory, user ID |
| **At least one thread** | the unit the scheduler actually runs (note 03) |

### 2.2 Memory layout of a process (64-bit Linux, simplified)
```
 high addresses
┌──────────────────────────┐  ← kernel space (not accessible in user mode)
├──────────────────────────┤
│ stack  (grows ↓)         │  local variables, return addresses, one per thread (8 MB default main)
│          ↓               │
│                          │
│ memory-mapped region     │  shared libraries (libc.so), mmap() files, big malloc()s, thread stacks
│                          │
│          ↑               │
│ heap   (grows ↑)         │  malloc/new (small allocations), top = "program break" (brk)
├──────────────────────────┤
│ .bss   (zero-initialised globals)   int counter;          │
│ .data  (initialised globals)        int limit = 10;       │
│ .rodata (constants, string literals) "hello"              │
│ .text  (machine code, read-only + executable)             │
└──────────────────────────┘
 low addresses   (see it live: cat /proc/<pid>/maps)
```
`static int x;` → .bss · `static int y = 5;` → .data · `char *s = "hi";` → `s` on the stack, `"hi"` in .rodata · `malloc(100)` → heap.

### 2.3 PCB (Process Control Block): the kernel's card for each process
Linux calls it `struct task_struct`. It holds:
- PID, parent PID, state, priority / scheduling info
- saved CPU registers (when not running)
- memory info: pointer to page tables (`mm_struct`), memory regions
- open files table, current directory, signal handlers and pending signals
- accounting: CPU time used, limits (`ulimit`), user/group IDs

### 2.4 Process states
```
           admitted                    scheduler dispatch
  NEW ─────────────► READY ◄──────────────────────────────┐
                      │  ▲                                  │
         dispatch     │  │ preempted (timer interrupt /     │
                      ▼  │ higher-priority task arrives)    │
                    RUNNING ─────────────────────────────── ┘
                      │   │
   waits for I/O,     │   │ exit()
   lock, sleep()      ▼   ▼
                  WAITING  TERMINATED (zombie until parent calls wait(); explained in §2.7)
                  (BLOCKED)
                      │
     I/O done / event │  → back to READY (NOT directly to running)
```
| Transition | Caused by |
|---|---|
| Ready → Running | the scheduler picks it |
| Running → Ready | preemption: time slice used up, or a higher-priority process woke up |
| Running → Waiting | it asks for something slow: `read()` from disk/network, `sleep()`, waiting on a lock |
| Waiting → Ready | the event happened (I/O interrupt, lock released) |
| Running → Terminated | `exit()` or killed by a signal |
Linux `ps` letters: `R` running/runnable, `S` interruptible sleep, `D` uninterruptible sleep (usually disk I/O; can't be killed), `T` stopped, `Z` zombie.

### 2.5 Context switch: step by step
Switching the CPU from process A to process B:
```
1. Something enters the kernel while A is running (timer interrupt, or A blocks in a syscall)
2. Save A's CPU state (registers, PC, SP) into A's PCB / kernel stack
3. Scheduler picks B
4. Switch address space: load B's page-table base (CR3 on x86, TTBR0 on ARM)
   → TLB entries of A are useless (unless tagged with PCID/ASID)
5. Restore B's registers from B's PCB, switch to B's kernel stack
6. Return to user mode → B continues where it left off
```
| Cost | Typical |
|---|---|
| Direct (save/restore, scheduler) | **~1–3 µs** |
| Indirect (cold caches, TLB misses, branch predictor) after the switch | can be **larger** than the direct cost, up to tens of µs of slowdown |
| Thread switch inside the same process | cheaper: **no address-space switch**, TLB and caches stay warm |
[HFT]: a trading thread **pinned to an isolated core** never context switches (note 13).

### 2.6 Process creation: fork, exec, wait, exit
```c
pid_t pid = fork();                 // ONE call, returns TWICE
if (pid < 0)      { perror("fork"); }
else if (pid == 0) {                // child
    execlp("ls", "ls", "-l", NULL); // replace the child's program with /bin/ls
    _exit(127);                     // only reached if exec FAILED
} else {                            // parent: pid = child's PID
    int status;
    waitpid(pid, &status, 0);       // wait for child, collect exit status
    if (WIFEXITED(status)) printf("exit code %d\n", WEXITSTATUS(status));
}
```
| Call | What it does |
|---|---|
| `fork()` | creates a **copy** of the calling process. Returns **0 in the child**, the **child's PID in the parent**, −1 on error. Memory is shared **copy-on-write** (only copied when someone writes; [COA 07 §2.10](../Computer%20Organization%20and%20Architecture/07-Virtual-Memory-and-TLB.md)) |
| `exec*()` | **replaces** the current program (code, data, heap, stack) with a new one. Same PID, open fds are kept (unless `O_CLOEXEC`). Does **not** return on success |
| `wait()/waitpid()` | parent blocks until a child exits, gets its exit status, and lets the kernel free the child's PCB |
| `exit()` | flushes stdio buffers, runs `atexit` handlers, then `_exit()` (the syscall) |

**What the child inherits from fork:** a copy of memory (CoW), **open file descriptors (sharing the same file offset!)**, signal handlers, environment, current directory. **Not inherited:** PID, pending signals, timers, file locks (`fcntl`); **only the calling thread** is copied (note 03).

**Why fork + exec as two steps, instead of one "spawn"?**

> **What "spawn" is:** a **single call** that creates a new process already running a different program: Windows `CreateProcess()` (Windows has no fork), POSIX `posix_spawn()`, Python `subprocess.Popen`, Java `ProcessBuilder`.
> ```
> SPAWN (one step):  parent ──spawn("ls")──► new process running ls
> UNIX (two steps):  parent ──fork()──► child = COPY of parent ──(any code here)──► exec("ls") ──► child runs ls
> ```
> **Why two steps:** in the gap between fork and exec, the child is a **separate process still running the parent's code**. It can change **its own** settings with ordinary syscalls (`dup2`, `close`, `chdir`, `setrlimit`, `setuid`, signal masks), and those settings **survive `exec`** (fds, cwd, limits and uid are kept). The new program starts already configured, without knowing it.
>
> **Example: the shell runs `ls -l > out.txt`:**
> ```c
> pid_t pid = fork();
> if (pid == 0) {                                          // child, still shell code
>     int fd = open("out.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
>     dup2(fd, 1); close(fd);                              // fd 1 (stdout) → out.txt
>     execlp("ls", "ls", "-l", NULL);                      // ls writes to fd 1 = the file
>     _exit(127);
> }
> waitpid(pid, NULL, 0);                                   // shell waits, then prints the prompt
> ```
> ```
> child fd table before dup2:  0→tty 1→tty 2→tty 3→out.txt
>                after dup2:   0→tty 1→out.txt 2→tty (3 closed)
> ```
> - `ls` just calls `write(1, …)` and has no idea it was redirected.
> - Why in the child, not in the shell before fork? `dup2` on the shell itself would send the **shell's own** output (your prompt) into the file.
> - Pipes work the same way: `ls | wc` = `pipe()` + two forks; child 1 `dup2(write_end, 1)` + exec ls, child 2 `dup2(read_end, 0)` + exec wc.
> - With one spawn call, every possible setup must be a **parameter** (CreateProcess has ~10 args + `STARTUPINFO`; posix_spawn has file-action and attribute objects). With fork + exec, the setup is **ordinary code**, so anything is possible.
>
> **Downside and alternatives:** fork copies the **page tables** (CoW avoids copying pages, not tables), only for exec to discard them. A 32 GB parent with 4 KB pages has 8 M PTEs × 8 B ≈ **64 MB** of tables ⇒ tens of ms wasted.
> | Option | How | Use |
> |---|---|---|
> | `fork` + `exec` | full CoW clone, then replace | flexible; fine for small parents (shells) |
> | `vfork` + `exec` | child **borrows** the parent's memory; the parent is paused until exec/_exit; no copy | the child must do almost nothing but exec |
> | **`posix_spawn`** | one call; glibc uses `clone(CLONE_VM \| CLONE_VFORK)` internally | big parents (JVMs, databases) launching helper programs |

#### 2.6.1 How Windows creates processes (no fork): `CreateProcess`
The Win32 API has **no `fork()`**. One spawn-style call builds a **fresh** process directly from an `.exe`:
```c
STARTUPINFOA si = { sizeof(si) }; PROCESS_INFORMATION pi;
CreateProcessA(NULL, "notepad.exe file.txt", NULL, NULL,
               FALSE,            // inherit handles? (≈ inheriting fds)
               0, NULL, NULL,    // flags (CREATE_SUSPENDED…), environment, current dir
               &si, &pi);        // std handles in, process/thread handles out
WaitForSingleObject(pi.hProcess, INFINITE);       // ≈ waitpid
DWORD code; GetExitCodeProcess(pi.hProcess, &code);
CloseHandle(pi.hProcess); CloseHandle(pi.hThread);
```
**Inside (`NtCreateUserProcess`), step by step:**
```
1. Open the .exe, validate the PE format, create a section object (image ready to map)
2. Create the process object (EPROCESS): a brand-NEW empty address space (no copy of the parent),
   map the exe + ntdll.dll, build the handle table (copy ONLY inheritable handles, if asked), create the PEB
3. Create the first thread (ETHREAD): stack, TEB, initial context → loader start routine
4. Register the new process with csrss.exe (Windows subsystem)
5. Start the thread: the ntdll loader maps DLLs (kernel32…), runs their init, jumps to main()/WinMain()
6. Return process + thread handles and PID/TID to the parent
```
| | Unix fork + exec | Windows CreateProcess |
|---|---|---|
| Steps | 2 (clone, then replace) | **1** (build from exe) |
| Child memory | CoW copy of the parent, then replaced | fresh address space from the exe |
| Inherited | all fds (unless CLOEXEC), cwd, env, limits | only handles marked inheritable (if requested); env, cwd |
| Redirection | `dup2` in the child before exec | pass handles in **`STARTUPINFO`** (`hStdOutput`… + `STARTF_USESTDHANDLES`) |
| Cost | cheap for small parents | heavier (often ms: image mapping, DLL loading, security/AV checks) ⇒ Windows favours **threads / thread pools** |
**Unix → Windows mapping:**
- `waitpid` → `WaitForSingleObject` + `GetExitCodeProcess`.
- **Zombie** → the process object lives on **while any handle to it is open** (exit code kept). Close your handles or they leak.
- No orphan re-parenting: there's no reaping duty, so Windows just records the parent PID.
- `pipe`+`dup2` → `CreatePipe` + `STARTUPINFO` handles.
- Signals → none in the Unix sense (`TerminateProcess` ≈ SIGKILL, console Ctrl events).
- fd → **handle**.

**Can Windows fork at all?**
- The NT kernel can clone internally (old POSIX subsystem, `RtlCloneUserProcess` for process snapshots), but Win32 apps can't use it.
- **Cygwin** emulates fork by copying memory manually (slow). **WSL 2** runs a real Linux kernel, so fork is real there.
- Practical effect: Python `multiprocessing` on Windows (and macOS by default) uses the **spawn** start method: a fresh interpreter **re-imports your script**, which is why process-starting code must be under `if __name__ == "__main__":`.

### 2.7 Zombie and orphan

> **What "zombie until the parent calls `wait()`" means: a process dies in two steps.**
> ```
> Step 1: child calls exit() → kernel frees almost everything, BUT keeps a tiny record
>                              (PID + exit status + CPU usage)  ⇒ child is a ZOMBIE ("dead, not buried")
> Step 2: parent calls wait() → kernel gives that record to the parent, THEN deletes it ⇒ fully gone
> ```
> **Why keep a record?** So the parent can learn *how* the child ended (exit 0? exit 1? killed by a signal?), and so the PID isn't reused while the parent might still `waitpid()` on it. Analogy: an employee leaves (exit). The desk and laptop are taken back at once (memory, fds), but HR keeps a one-page final report until the manager (parent) collects it (`wait`).
> | Freed at `exit()` | Kept until `wait()` |
> |---|---|
> | memory, page tables, open files/sockets, threads, timers | PID, exit status, CPU-usage stats, a small `task_struct` entry |
> A zombie uses no CPU or memory; its only cost is the **PID slot**. `ps` shows it as `Z … <defunct>`.
> ```c
> pid_t pid = fork();
> if (pid == 0) exit(7);          // child exits at once → zombie for the next 30 s
> sleep(30);                      // parent not waiting yet
> int st; waitpid(pid, &st, 0);   // reaped now; WEXITSTATUS(st) == 7
> ```
> **How parents reap:** `wait`/`waitpid` (blocking) · a **SIGCHLD** handler with `while (waitpid(-1, &st, WNOHANG) > 0);` · `signal(SIGCHLD, SIG_IGN)` = auto-reap (no zombies, no statuses). The shell reaps every command; `$?` is the status it collected.
> **Never reaped:** zombies pile up until the PID limit, then `fork()` fails with `EAGAIN`. E.g. 100 forks/s without `wait`, `pid_max` 32,768 ⇒ broken after ≈ **328 s**. If the parent exits, init (PID 1) adopts and reaps them, so **kill or fix the parent**, not the zombie. [AMZN] In a container your app may be PID 1 ⇒ use `tini` / `docker run --init` to reap.
> Find them: `ps -eo pid,ppid,stat,cmd | grep ' Z'` (PPID = the parent to fix).
| | Zombie | Orphan |
|---|---|---|
| What | child **exited**, but the parent hasn't called `wait()` yet | **parent exited** while the child is still running |
| Why it exists | the kernel keeps the child's PID + exit status so the parent can read them | — |
| What happens | stays as `Z` in `ps`, holding a PID slot | re-parented to **init (PID 1)** or a "subreaper", which waits on it automatically |
| Danger | many zombies ⇒ PID table exhaustion | usually harmless |
| Fix | parent calls `wait`, or handles `SIGCHLD`, or sets `SIGCHLD` to `SIG_IGN` (auto-reap) | — |
You **can't kill a zombie** (it's already dead). Kill or fix its **parent**.

### 2.8 Process vs program vs thread (quick)
| | Program | Process | Thread |
|---|---|---|---|
| What | file on disk | running instance | execution path inside a process |
| Memory | none | own address space | shares the process's address space |
| Created by | compiler | `fork`/`exec` | `pthread_create` / `clone` |

---

## 3. Worked Numericals (fork puzzles are asked everywhere)

**N1. Forks in a loop.**
```c
for (int i = 0; i < 3; i++) fork();
printf("x\n");
```
Each `fork()` doubles the number of processes: 1 → 2 → 4 → 8. Every process prints once ⇒ **8 lines**. Child processes created = 8 − 1 = **7**. General: n forks in sequence ⇒ **2ⁿ** processes, **2ⁿ − 1** children.

**N2. Print inside the loop.**
```c
for (int i = 0; i < 3; i++) { fork(); printf("x\n"); }
```
- After the fork in iteration i there are 2^(i+1) processes, each printing once: 2 + 4 + 8 = **14 lines** (output to a terminal).
- **Trap:** run it as `./a.out > file`. stdout to a file is **fully buffered**, so the "x\n" lines sit in the buffer and get **copied by fork**. Each of the 8 final processes ends with exactly 3 lines in its buffer ⇒ **24 lines**. Fix: `fflush(stdout)` before `fork()`.

**N3. Short-circuit puzzle.**
```c
fork() && fork() || fork();
printf("x\n");
```
Step by step (P0 = original):
1. P0 calls fork #1 → creates C1. In C1 it returns 0, so `0 && …` is false and C1 skips fork #2, then evaluates `|| fork()` → **C1 creates C2**.
2. In P0, fork #1 returns non-zero, so P0 evaluates fork #2 → **creates C3**. In P0 fork #2 is non-zero ⇒ `true || …` short-circuits, so no fork #3.
3. In C3, fork #2 returned 0 ⇒ the `&&` is false ⇒ evaluate `|| fork()` → **C3 creates C4**.
Processes: P0, C1, C2, C3, C4 ⇒ **5 processes, 5 lines**.

**N4. Buffering + fork.**
```c
printf("A");  fork();  printf("B\n");
```
"A" has no newline ⇒ it stays in the stdio buffer, and fork copies the buffer ⇒ both processes print "AB\n" ⇒ output **AB AB** (two lines), not "A B B".

**N5. Context-switch overhead.** A server does 20,000 context switches/s per core at 3 µs each (direct + indirect).
Overhead = 20,000 × 3 µs = 60 ms per second = **6 %** of each core.

**Try it:** how many times is "hi" printed?
```c
int main() { if (fork() == 0) { fork(); } printf("hi\n"); }
```
<details><summary>Answer</summary>P0 forks C1. Only C1 (fork returned 0) forks again → C2. Processes: P0, C1, C2 ⇒ <b>3</b>.</details>

---

## 4. Interview Questions

### Basic
- **Q: Program vs process?** A program is a passive file; a process is an active instance with an address space, registers, resources and a PCB.
- **Q: What's in a PCB?** §2.3: PID, state, registers, memory/page-table pointer, open files, signals, scheduling and accounting info.
- **Q: Process states?** New, ready, running, waiting/blocked, terminated (§2.4). Follow-up: *"Can a waiting process go directly to running?"* No, it goes to ready first.
- **Q: What does fork return?** 0 to the child, the child's PID to the parent, −1 on failure.

### Intermediate
- **Q: Zombie vs orphan? How do you remove a zombie?** §2.7. You can't kill it. Make the parent `wait()`, or kill the parent so init adopts and reaps it.
- **Q: Unix starts programs with fork + exec. Other systems use a single "spawn" call (Windows `CreateProcess`, `posix_spawn`). Why two steps?** Between fork and exec, the child (still running the parent's code) can configure itself with ordinary syscalls: `dup2` for redirection, close fds, `chdir`, limits, uid. These survive exec, so the new program starts configured. Example: `ls > out.txt` = fork → child `open` + `dup2(fd, 1)` → exec ls (§2.6 box). Follow-up: *"What's the downside?"* fork copies page tables only to discard them ⇒ big processes use `posix_spawn`/`vfork`.
- **Q: Windows has no fork. How does it create processes?** `CreateProcess`: one call that builds a fresh address space from the exe, inherits only handles marked inheritable, sets std handles via `STARTUPINFO`, creates the first thread, and the loader maps DLLs and calls main (§2.6.1). Follow-up: *"Why do Windows programs prefer threads?"* Process creation is expensive and there's no cheap CoW fork.
- **Q: Fork copies the whole address space, so isn't it slow?** No, copy-on-write: page tables are copied with pages marked read-only, and pages are copied only on a write. Cost ∝ page-table size, not memory size.
- **Q: What exactly happens in a context switch? What does it cost?** §2.5: save/restore, switch page tables, ~1–3 µs direct plus cache/TLB warm-up.

### Advanced / killer follow-ups
- **Q: Parent and child both write to the same fd after fork. What happens?** They share the **open-file description**, including the **file offset**, so writes don't overwrite each other: they interleave (note 10 three-table diagram).
- **Q: The fork-in-loop output differs when redirected to a file. Why?** stdio buffering: full buffering to a file means the unflushed buffer is duplicated by fork (N2).
- **Q: What happens if a multithreaded process calls fork?** Only the calling thread exists in the child. Locks held by other threads stay locked forever ⇒ only call async-signal-safe functions, then `exec` (note 03).
- **Q [HFT]: How do you avoid context switches on a latency-critical thread?** Pin it to an isolated core (`isolcpus`/`nohz_full`, affinity), never block (busy-poll), avoid syscalls/page faults, give it real-time priority if needed (note 13).
- **Q: What is a `D` state process? Why can't `kill -9` remove it?** Uninterruptible sleep (waiting on I/O inside the kernel). Signals are delivered only when it returns from the kernel.

---

## 5. Common Traps
- Saying the child starts from `main()`. It continues from the **return of `fork()`**.
- Forgetting fork returns twice, and short-circuit evaluation in `&&`/`||` puzzles.
- Ignoring stdio buffering in fork-print questions.
- "fork copies all memory immediately". No: copy-on-write.
- `exec` creates a new process. No: same PID, new program.
- Trying to `kill` a zombie.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Process | program in execution: address space + registers + resources + ≥1 thread |
| Layout | text, rodata, data, bss, heap ↑, mmap region, stack ↓, kernel on top |
| PCB | `task_struct`: PID, state, regs, mm, files, signals, sched info |
| States | new → ready ⇄ running → waiting → ready; terminated (zombie until wait) |
| Context switch | save regs → pick next → switch page tables → restore; ~1–3 µs + cache/TLB effects |
| fork | returns 0 / child PID / −1; CoW; shares fd offsets |
| exec | replaces the program, same PID, doesn't return on success |
| Windows | no fork: `CreateProcess` (fresh address space from exe, inheritable handles, STARTUPINFO redirection); wait = WaitForSingleObject; "zombie" = process object alive while handles open |
| fork+exec vs spawn | gap lets the child `dup2`/close/chdir before exec (shell redirection); spawn = one call (CreateProcess, posix_spawn); big parents use posix_spawn/vfork |
| wait | reaps child, gets status; no wait ⇒ zombie |
| Orphan | adopted by init (PID 1) |
| Fork count | n sequential forks ⇒ 2ⁿ processes; watch `&&`/`||` and buffering |
