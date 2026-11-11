# 12. IPC & Signals

> **Asked at:** [HFT] fastest IPC between two processes on one box (shared memory + lock-free ring), cost of pipes/sockets · [AMZN] [DESHAW] IPC mechanisms compared, pipes, signals, zombie reaping with SIGCHLD · [QCOM] [NVIDIA] shared memory, Android Binder · [APPLE] Mach ports, XPC.
> **Prerequisites:** [02 Processes](02-Processes.md), [09 Virtual Memory](09-Virtual-Memory-Page-Faults-and-Replacement.md) (mmap), [05 Synchronization](05-Synchronization-Primitives.md).

---

## 1. Intuition
Processes live in separate houses (address spaces), so they can't just read each other's memory. To communicate they can:
- **send letters through the post office** (the kernel copies the message): pipes, message queues, sockets. This is **message passing**;
- **share a common room** that both houses open into: **shared memory**. It's the fastest, but they must agree on who uses the room when (synchronization);
- **ring a doorbell**: signals, a tiny notification that carries just a number.

---

## 2. Deep Dive

### 2.1 Two models
| | Message passing | Shared memory |
|---|---|---|
| How | the kernel copies data from the sender to the receiver | both processes map the same physical pages |
| Syscalls per message | ≥ 1 send + 1 receive | **none** after setup |
| Copies | 2 (user → kernel → user) | **0** |
| Synchronization | built in (receive blocks until data) | **your job** (atomics, process-shared mutex/semaphore) |
| Latency (same machine) | ~2–10 µs | **~50–200 ns** (one cache-line transfer between cores) |
| Safety | isolated, simple | a bug in one process can corrupt the other's data |

### 2.2 Pipes
```c
int fds[2];
pipe(fds);                      // fds[0] = read end, fds[1] = write end
if (fork() == 0) {              // child: writer
    close(fds[0]);
    write(fds[1], "hello", 5);
    close(fds[1]);
    _exit(0);
}
close(fds[1]);                  // parent: reader. Close the unused write end!
char buf[16]; int n = read(fds[0], buf, sizeof buf);   // blocks until data or EOF
```
- **Unidirectional** byte stream with a kernel buffer (64 KB default on Linux). Two pipes are needed for two-way traffic.
- `read` blocks when empty; `write` blocks when full. Writes ≤ `PIPE_BUF` (4096 B) are **atomic** (not interleaved with other writers).
- `read` returns **0 (EOF)** only when **all write ends are closed**. Forgetting to close unused write ends ⇒ the reader waits forever (classic bug).
- Writing to a pipe with no readers ⇒ **SIGPIPE** (default: kill), or `EPIPE` if the signal is ignored.
- Only between related processes (inherited via fork). The shell's `ls | wc` = `pipe` + `fork` ×2 + `dup2` + `exec`.
- **Named pipe (FIFO):** `mkfifo /tmp/f` creates a filesystem name, so **unrelated** processes can open it. Same semantics otherwise.

### 2.3 Message queues
POSIX `mq_open/mq_send/mq_receive` (or System V `msgget/msgsnd/msgrcv`): the kernel keeps **discrete messages** (not a byte stream) with **priorities**. `mq_notify` can signal arrival. Message boundaries are preserved.

### 2.4 Shared memory [HFT]
```c
// Process A (creator)
int fd = shm_open("/md_ring", O_CREAT | O_RDWR, 0600);   // object in /dev/shm (tmpfs)
ftruncate(fd, SIZE);
void *p = mmap(NULL, SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
// Process B: shm_open("/md_ring", O_RDWR, 0) + the same mmap → same physical pages
```
- After setup, a write by A is visible to B through the **cache coherence** hardware. No kernel involvement, no copies.
- **Synchronization options:**
  - `std::atomic` in the shared region (lock-free, address-free atomics). The **SPSC ring buffer** from [COA 10 §3.8](../Computer%20Organization%20and%20Architecture/10-Cache-Coherence-and-Memory-Consistency.md) works across processes;
  - `pthread_mutex` with `PTHREAD_PROCESS_SHARED` (+ robust mutexes so a crashed owner doesn't leave it locked forever);
  - POSIX semaphores (`sem_init(…, pshared = 1)` in shared memory, or named `sem_open`).
- **Pointers:** don't store raw pointers in shared memory; the region may be mapped at different addresses in each process. Use **offsets**.
- **HFT pattern:** feed handler process → shared-memory ring buffer → strategy processes (each busy-polling the ring); huge pages + `mlock` for the region. Process separation gives fault isolation at almost thread-level speed.
- System V equivalent: `shmget/shmat`. Simplest: `mmap(MAP_SHARED | MAP_ANONYMOUS)` before `fork` (parent-child only).

### 2.5 Sockets
| | Unix domain socket (`AF_UNIX`) | TCP loopback (`127.0.0.1`) |
|---|---|---|
| Scope | same machine | same machine (or network) |
| Path | the kernel copies between socket buffers; no protocol stack | full TCP/IP stack (checksums, ACKs, …) |
| Latency | lower (~2–5 µs) | higher |
| Extras | can pass **file descriptors** (`SCM_RIGHTS`) and credentials | portable to remote hosts |
Used by Docker (`/var/run/docker.sock`), X11/Wayland, databases (local connections), systemd.

### 2.6 Platform-specific IPC [APPLE][QCOM]
- **Android Binder** [QCOM]: a kernel driver for RPC between apps and system services. It copies data **once** (into a buffer mmapped into the receiver), carries the caller's identity, and manages object references.
- **Mach ports** [APPLE]: XNU's core IPC. Messages are sent to port rights; large data is transferred with **copy-on-write** VM tricks. **XPC** (higher level) is used for app ↔ service communication and privilege separation.
- **D-Bus** (Linux desktops/embedded): a message bus on top of Unix sockets.

### 2.7 Signals
A signal is an **asynchronous notification** to a process: just a number, no data.
| Signal | Default action | Typical cause |
|---|---|---|
| `SIGINT` (2) | terminate | Ctrl-C |
| `SIGTERM` (15) | terminate | `kill pid` (polite shutdown request) |
| `SIGKILL` (9) | terminate | `kill -9`: **can't be caught, blocked or ignored** |
| `SIGSEGV` (11) | core dump | invalid memory access |
| `SIGFPE` | core dump | integer divide by zero |
| `SIGBUS` | core dump | bad alignment / mmapped file truncated |
| `SIGCHLD` | ignore | a child stopped or exited |
| `SIGPIPE` | terminate | write to a pipe/socket with no reader |
| `SIGALRM` | terminate | `alarm()` timer expired |
| `SIGSTOP` / `SIGCONT` | stop / continue | job control; SIGSTOP can't be caught |
| `SIGUSR1/2` | terminate | user-defined (e.g. "reload config", "dump stats") |

**How delivery works:**
```
1. kill(pid, SIGTERM) or a kernel event → the kernel sets a bit in the target's PENDING set
2. If the signal is BLOCKED (in the signal mask) → it stays pending
3. When the target next returns from kernel mode to user mode (after a syscall, interrupt,
   or being scheduled), the kernel checks pending & ~blocked
4. Action: default (terminate/core/ignore/stop) OR run the user's HANDLER:
   the kernel builds a frame on the user stack so the process jumps into the handler,
   then returns via sigreturn to where it was interrupted
```
- Standard signals **don't queue**: two `SIGCHLD`s arriving while one is pending = one delivery ⇒ in the handler, reap in a loop: `while (waitpid(-1, &st, WNOHANG) > 0);`.
- A blocking syscall interrupted by a handler returns **`EINTR`** (unless `SA_RESTART`) ⇒ robust code retries.
- **Install handlers with `sigaction`** (not `signal()`, whose semantics vary).

**Async-signal-safety (killer topic):** a handler can run **between any two instructions** of the main program, including inside `malloc` or `printf` while they hold internal locks. If the handler calls `malloc`/`printf` too ⇒ deadlock or corrupted heap.
- Only call **async-signal-safe** functions in handlers: `write`, `_exit`, `signal`-related calls, `sem_post`… (a list in `man 7 signal-safety`). **Not** `printf`, `malloc`, `new`, `std::cout`, mutex lock.
- Best practice: the handler only sets `volatile sig_atomic_t flag = 1;` (or writes a byte to a **self-pipe**), and the main loop does the real work. Modern Linux: **`signalfd`** turns signals into fd events for epoll ⇒ no async handlers at all.
- **Multithreaded programs:** process-directed signals go to **any** thread not blocking them ⇒ block all signals in every thread (`pthread_sigmask`) and dedicate one thread to `sigwait()` (or use `signalfd`).

---

## 3. Worked Numericals

**N1. Copies and syscalls per message.** Send 1 M messages of 64 B between two processes.
| Mechanism | Syscalls | Copies | Est. per message | Total |
|---|---|---|---|---|
| Pipe | 2 M (`write` + `read`) | 2 M × 2 | ~4 µs | **~4 s** |
| Unix socket | 2 M | 2 per message | ~3 µs | ~3 s |
| Shared-memory SPSC ring | 0 | 1 store + 1 load of the line | ~0.1 µs | **~0.1 s** |
⇒ shared memory ≈ **40×** faster here, and with no kernel jitter.

**N2. Pipe capacity behaviour.** The pipe buffer is 64 KB. The writer writes 100 KB in one `write()` while nobody reads.
The first 64 KB fills the buffer; the writer **blocks** with 36 KB pending. As the reader consumes, the writer continues. (With `O_NONBLOCK`: a partial write of 65,536 B is returned, then `EAGAIN`.) The write isn't atomic (> PIPE_BUF), so another writer's data could interleave.

**N3. SIGCHLD coalescing.** 5 children exit within 1 ms while the parent is busy (SIGCHLD blocked/pending). How many times does the handler run? Possibly **once** (standard signals don't queue) ⇒ a handler doing one `wait()` leaves **4 zombies**. A `while (waitpid(-1, …, WNOHANG) > 0)` loop reaps all 5.

**Try it:** a parent creates a pipe, forks, and the child reads until EOF. The parent writes 10 bytes and closes its write end, but the child **forgot to close its own copy** of the write end. What happens?
<details><summary>Answer</summary>The child reads the 10 bytes, then <code>read</code> <b>blocks forever</b>: a write end is still open (its own), so EOF never comes. Always close unused pipe ends right after fork.</details>

---

## 4. Interview Questions

### Basic
- **Q: What IPC mechanisms exist?** Pipes, FIFOs, message queues, shared memory, sockets, signals, mmapped files (+ platform ones: Binder, Mach ports).
- **Q: Shared memory vs message passing?** §2.1 table.
- **Q: Pipe vs named pipe?** Anonymous pipes need a related process (inherited fds); a FIFO has a path, so unrelated processes can use it.
- **Q: What is a signal? Which can't be caught?** An async notification; SIGKILL and SIGSTOP can't be caught.

### Intermediate
- **Q: What is the fastest IPC on one machine? Why?** Shared memory: no syscalls or copies after setup. You add synchronization (atomics/ring buffer).
- **Q: Why does my pipe reader never see EOF?** An unused write end is still open somewhere (often in the reader itself).
- **Q: What is SIGPIPE and how do servers handle it?** A write to a closed pipe/socket. Ignore it (`signal(SIGPIPE, SIG_IGN)` or `MSG_NOSIGNAL`) and handle `EPIPE`.
- **Q: What happens to a blocking `read()` when a signal arrives?** The handler runs; `read` returns `EINTR` unless `SA_RESTART` ⇒ retry.

### Advanced / killer follow-ups
- **Q: Why can't you call `printf` or `malloc` in a signal handler?** They aren't async-signal-safe: they may hold internal locks or be mid-update when interrupted ⇒ deadlock or corruption. Set a `sig_atomic_t` flag / self-pipe / `signalfd`.
- **Q: How are signals handled in a multithreaded process?** Delivered to any unblocking thread ⇒ block in all threads, `sigwait` in a dedicated thread or `signalfd`.
- **Q [HFT]: Design market-data distribution from one feed process to 5 strategy processes on the same box.** Shared-memory ring buffer(s) (SPMC or one SPSC ring per consumer), `alignas(64)` indices, huge pages + `mlock`, consumers busy-poll on pinned cores, sequence numbers to detect overruns, offsets instead of pointers.
- **Q: Can you put a `std::mutex` in shared memory between processes?** Not portably. Use `pthread_mutex` with `PTHREAD_PROCESS_SHARED` (+ robust), or lock-free atomics, which work across processes when lock-free.
- **Q [QCOM]: What is Binder and why does Android use it instead of sockets?** Single-copy RPC with caller identity (UID/PID) for security checks, object references, and death notifications.

---

## 5. Common Traps
- Forgetting to close unused pipe ends.
- Storing raw pointers in shared memory.
- Assuming signals queue (standard ones don't).
- Non-async-signal-safe calls in handlers.
- Using `signal()` instead of `sigaction()`.
- Thinking shared memory needs no synchronization.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Models | message passing (kernel copies, built-in sync) vs shared memory (0 copies, DIY sync) |
| Pipe | unidirectional, 64 KB buffer, EOF when all writers close, SIGPIPE, atomic ≤ 4 KB |
| FIFO | named pipe for unrelated processes |
| Message queue | discrete messages + priorities |
| Shared memory | `shm_open` + `ftruncate` + `mmap(MAP_SHARED)`; atomics / pshared mutex; offsets not pointers |
| Sockets | AF_UNIX (fast, fd passing) vs TCP loopback |
| Platform | Binder (Android), Mach ports / XPC (Apple) |
| Signals | pending bit → delivered on return to user mode; SIGKILL/SIGSTOP uncatchable |
| Handlers | `sigaction`; only async-signal-safe calls; flag / self-pipe / signalfd |
| SIGCHLD | coalesces ⇒ `waitpid(-1, WNOHANG)` loop |
| HFT | shared-memory SPSC ring, busy-poll, ~100 ns |
