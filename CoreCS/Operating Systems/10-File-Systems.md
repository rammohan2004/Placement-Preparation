# 10. File Systems

> **Asked at:** [AMZN] [DESHAW] inodes, hard vs soft links, what happens on `open`/`read`, journaling, durability with `fsync` · [APPLE] APFS (copy-on-write, snapshots, clones) · [HFT] page cache, `O_DIRECT`, logging without stalling · [NVIDIA] [QCOM] VFS, device files, `/proc` `/sys`.
> **Prerequisites:** [02 Processes](02-Processes.md) (fork shares fds), [09 Virtual Memory](09-Virtual-Memory-Page-Faults-and-Replacement.md) (page cache, mmap).

---

## 1. Intuition
A disk is a huge array of numbered **blocks** (4 KB each). A file system is the **library catalogue** that turns that array into named files and folders:
- the **inode** is the catalogue card for a book: who owns it, size, permissions, and *which shelves (blocks) its pages are on*, but **not its name**;
- a **directory** is a list of (name → card number) entries;
- the **journal** is the librarian's notebook: write what you're *about* to change first, so after a power cut you can finish or undo it.

---

## 2. Deep Dive

### 2.1 File descriptors: the three tables (very commonly asked)
```
 Process A fd table          System-wide open-file table          Inode table (in memory)
 ┌────┬──────────┐           ┌──────────────────────────┐          ┌───────────────────────┐
 │ 0  │ → tty    │           │ OFD #1: offset = 120      │ ───────► │ inode 4711 (log.txt)  │
 │ 1  │ → tty    │     ┌───► │         flags = O_WRONLY  │          │ size, owner, blocks…  │
 │ 3  │ ─────────┼─────┘     │         refcount = 2      │          └───────────────────────┘
 └────┴──────────┘     ┌───► │                           │                    ▲
 Process B (fork child)│     ├──────────────────────────┤                    │
 ┌────┬──────────┐     │     │ OFD #2: offset = 0        │ ───────────────────┘
 │ 3  │ ─────────┼─────┘     │ (A opened log.txt AGAIN)  │
 └────┴──────────┘           └──────────────────────────┘
```
| Table | One per | Holds |
|---|---|---|
| **fd table** | process | small integers 0, 1, 2, … → pointers to open-file descriptions |
| **Open-file description (OFD)** | each `open()` call | **current offset**, access mode, flags (O_APPEND…), refcount |
| **Inode** | each file | metadata + block map (shared by everyone who opened the file) |
Consequences:
- **fork / `dup` / `dup2`** ⇒ two fds point to the **same OFD** ⇒ they share the offset (parent and child writes interleave instead of overwriting).
- Two separate `open()`s of the same file ⇒ two OFDs ⇒ **independent offsets** (they can overwrite each other unless O_APPEND).
- `dup2(fd, 1)` is how the shell redirects stdout: fd 1 now points to the file's OFD.
- fd 0, 1, 2 = stdin, stdout, stderr. `open()` returns the **lowest free** fd number.

> **"Parent and child share one offset, but run in parallel on different cores. How can that work?"**
> `fork` **copies the fd table** but **shares the OFD**. The offset lives in the OFD, which is **one object in kernel memory**, reachable from every core (RAM is shared). Two cores *can* hit it at once, so it's an ordinary race, and the kernel solves it with a lock.
>
> | Item | After `fork` | Consequence |
> |---|---|---|
> | fd table | **copied** | child's `close(3)` doesn't close the parent's fd 3 |
> | OFD (offset, flags) | **shared** | same offset; `fcntl(F_SETFL, O_NONBLOCK)` in one affects the other |
> | inode, page cache | shared | same file data |
>
> Linux's OFD (`struct file`) has a mutex, **`f_pos_lock`**. Each `read`/`write` on a regular file does: *lock → pos = offset → transfer [pos, pos+n) → offset = pos + n → unlock*.
> ```
> Core 0: parent write(3,"AAAA",4)      Core 1: child write(3,"BBBB",4)
>   lock ✓, pos = 0 → bytes [0,4)          lock → BLOCKED
>   offset = 4, unlock  ─────────────►     lock ✓, pos = 4 → bytes [4,8), offset = 8, unlock
> File = "AAAABBBB" or "BBBBAAAA". Never "ABAB", never an overwrite.
> ```
> - Interleaving happens per **whole `write()` call**. The order isn't defined; *no overlap* is guaranteed. POSIX requires this for regular files; Linux has done it correctly since 3.14.
> - Shared-fd `read()`s **split the data**: each chunk goes to only one of parent/child.
> - The lock is taken only when the OFD is shared (refcount > 1).
> - **Two separate `open()`s = two OFDs, two offsets ⇒ both write at 0 ⇒ the last writer wins.** Fix: `O_APPEND` (each write moves to EOF atomically under the inode lock).
> - **`pread`/`pwrite`** take an explicit offset and never touch the shared one, so there's no offset contention between threads (databases use them).
> - Trap: **stdio buffers are user memory, duplicated by `fork`.** `printf("hi")` (no `\n`) then `fork` ⇒ "hi" is printed twice. stdio flushes in buffer-sized chunks, so lines can interleave mid-line.

### 2.2 Inode and directories
**Inode (index node)** contains: file type, permissions (rwx for user/group/other), owner UID/GID, size, timestamps (atime, mtime, ctime), **link count**, and **pointers to data blocks**. It does **not** contain the file name.
**Directory** = a special file containing entries (name → inode number). `.` = itself, `..` = parent.

**Path resolution for `/home/ram/a.txt`:**
```
root inode (#2, well-known) → read root dir blocks → find "home" → inode #N1
→ read inode N1 → read its dir blocks → find "ram" → inode #N2
→ read inode N2 → dir blocks → find "a.txt" → inode #N3 → read inode N3 → data blocks
```
Each step may need a disk read, so the kernel caches lookups in the **dentry cache** and **inode cache**.

> **"Is a directory also an inode? Is N1 the inode of `home`?"** Yes and yes.
> A directory is a file with its **own inode**, whose *type* field = directory. Its block pointers lead to blocks of **directory entries** (name → inode #), not to file bytes. **No inode stores its own name.** "home" is written inside **root's** data block as `"home" → N1`, which is why hard links work.
> ```
> inode 2 (DIR) "/"          → block 1000: "." → 2, ".." → 2, "home" → 131073 (N1), "etc" → 262145
> inode 131073 (DIR) /home   → block 5200: "." → 131073, ".." → 2, "ram" → 131200 (N2)
> inode 131200 (DIR) ram     → block 6100: "." → 131200, ".." → 131073, "a.txt" → 131305 (N3)
> inode 131305 (REGULAR, 3000 B) → block 7000: the file's bytes
> ```
> Per step: (1) read the **inode** (permissions + where its blocks are); (2) read the directory's **data block** and search it for the next name.
>
> **Inode number → disk location is arithmetic, not a search.** Inodes sit in a fixed on-disk **inode table**; ext4 splits it across *block groups*: group = (ino − 1) / inodes_per_group, index = (ino − 1) % inodes_per_group, byte offset = index × inode size. E.g. ino 131073, 8192 per group ⇒ group 16, slot 0.
> **Why root = #2:** 0 means "no inode" (empty entry); ext2/3/4 reserve 1 (historically the bad-blocks list). So the root is a fixed well-known number (`ls -id /` ⇒ `2 /`).
> **Directory link count** = 2 + number of subdirectories (its name in the parent, its own ".", and each child's "..").
>
> | Bit | File | Directory |
> |---|---|---|
> | r | read contents | **list** names |
> | w | modify contents | **create / delete / rename entries** |
> | x | execute | **traverse** (use it in a path) |
>
> Resolving `/home/ram/a.txt` needs **x** on `/`, `/home`, `/home/ram`. **Deleting a file needs w on its directory**, not on the file: unlink only edits the directory's data.

### 2.3 Hard links vs symbolic (soft) links
> Builds on §2.2: a directory is a list of (name → inode number). A **link** = a way to reach a file by a name. **Hard link** = *another name* for the same inode. **Soft (symbolic) link** = *a small separate file that contains a path* to another name.

#### 2.3.1 Hard link = just another directory entry
**Every ordinary file name is already a hard link.** Creating `a.txt` makes inode 131305 and adds the entry `"a.txt" → 131305` to the directory ⇒ link count = 1. `ln a.txt b.txt` adds **one more entry with the same inode number**. Nothing is copied.
```
$ echo hello > a.txt
$ ln a.txt b.txt
$ ls -li
131305 -rw-r--r-- 2 ram ram 6  a.txt     ← same inode number
131305 -rw-r--r-- 2 ram ram 6  b.txt     ← link count 2

directory /home/ram (data block)            inode 131305 (REGULAR, links = 2)
   "a.txt" → 131305 ──┐                      ──► block 7000: "hello\n"
   "b.txt" → 131305 ──┘
```
- The names are **equal**: there's no "original" and no "link". Writing via `b.txt` changes what `a.txt` shows (one inode, one set of data blocks and permissions).
- `rm a.txt` removes only that **entry**; the link count goes 2 → 1 and `cat b.txt` still prints `hello`. Data is freed only at link count 0 **and** no open fds.

#### 2.3.2 Soft (symbolic) link = a small file holding a path
`ln -s a.txt c.txt` creates a **new inode of type symlink** whose content is the text `a.txt`.
```
$ ln -s a.txt c.txt
$ ls -li c.txt
131400 lrwxrwxrwx 1 ram ram 5  c.txt -> a.txt    ← own inode, type 'l', size 5 = strlen("a.txt")

"c.txt" → 131400 (SYMLINK, content "a.txt") ··· by NAME ···► whatever "a.txt" currently is
```
When path resolution reaches a symlink inode, it **reads the stored path and continues the walk with it** (a relative path is taken from the symlink's own directory). So a symlink points to a **name**, not an inode. `stat` follows it; `lstat` describes the link itself; `open(..., O_NOFOLLOW)` refuses to follow it.

#### 2.3.3 One experiment shows every difference
```
start:   a.txt ─► inode 131305 "hello";  b.txt = hard link;  c.txt -> "a.txt"

$ rm a.txt
   b.txt → inode 131305, still "hello"          (links 2 → 1)
   c.txt → name "a.txt" not found → ENOENT      (DANGLING)

$ echo bye > a.txt       # NEW file, new inode 131500
   b.txt → still "hello"   (tied to inode 131305)
   c.txt → "bye"           (follows whatever is NAMED a.txt now)
```
**Why the restrictions in the table below exist:**
- **No hard links across file systems:** an entry stores only an inode **number**, and #131305 on disk A means nothing on disk B.
- **No hard links to directories:** they could create cycles (`find`/`du` loop forever, and `..` becomes ambiguous). Symlink cycles are possible but bounded: the kernel follows at most 40 per path (`ELOOP`), and `find` doesn't follow symlinks by default.

#### 2.3.4 Comparison table
| | Hard link (`ln a b`) | Symbolic link (`ln -s a b`) |
|---|---|---|
| What it is | another directory entry pointing to the **same inode** | a small file whose content is a **path** |
| Link count | increases the inode's count | doesn't change the target's count |
| Delete original name | data stays (count > 0) | link becomes **dangling** |
| Across file systems | ❌ (inode numbers are per-FS) | ✅ |
| To directories | ❌ (would create cycles) | ✅ |
| Own inode? | no (shares one) | yes |
**Deleting a file = `unlink()`** removes one name and decrements the link count. Blocks are freed only when the **link count = 0 AND no process has it open**. That's why disk space isn't freed when you delete a huge log a process still writes to (`lsof | grep deleted`), and why "unlink a temp file right after opening it" is a trick for auto-cleanup.

**Where each is used:**
| Hard links | Soft links |
|---|---|
| `mv` within one FS = link the new name + unlink the old ⇒ **instant, no data copied** (across file systems `mv` must copy) | shared-library versions: `libfoo.so → libfoo.so.1 → libfoo.so.1.2.3` |
| backup snapshots (`rsync --link-dest`): unchanged files are shared, not copied | `/usr/bin/python → python3.12`; release switching (`current → release-42`) |
| | pointing into another file system or disk |

### 2.4 Allocation methods: where a file's blocks live
| Method | How | Pros | Cons |
|---|---|---|---|
| **Contiguous** | file = start block + length | fast sequential + random access | external fragmentation; hard to grow (used in CD-ROMs) |
| **Linked** | each block points to the next | no external fragmentation, easy to grow | random access is O(n); a broken pointer loses the rest |
| **FAT** (linked, but pointers kept in a table) | File Allocation Table in memory: entry i = next block after i | random access by walking the table in RAM | table size ∝ disk size; FAT32 4 GB file limit (USB sticks, SD cards) |
| **Indexed (inode)** | the inode holds pointers: direct + single/double/triple indirect | fast random access, small files cheap | pointer overhead; big files need indirect reads |
| **Extents** (ext4, XFS, NTFS, APFS) | (start, length) runs instead of per-block pointers | compact for big contiguous files | — |

**Unix inode pointer layout (ext2/3 style):**
```
inode: [12 direct pointers] → 12 data blocks
       [single indirect]    → block of pointers → data blocks
       [double indirect]    → block of pointers → blocks of pointers → data blocks
       [triple indirect]    → three levels → data blocks
```

### 2.5 Free-space management
- **Bitmap:** 1 bit per block (1 = used). Easy to find contiguous runs. Size = blocks / 8 bytes.
- Free list / grouping / counting (runs of free blocks); B-trees of free extents (XFS, btrfs).

### 2.6 Crash consistency: journaling and copy-on-write
Appending one block to a file updates **three** things: the data block, the inode (size, pointer) and the free-space bitmap. A crash between these writes leaves the FS inconsistent (e.g. the bitmap says used but no inode points there ⇒ leaked; or the inode points at garbage).
- **fsck** (old way): scan the whole disk after a crash to fix inconsistencies ⇒ minutes to hours on big disks.
- **Journaling (write-ahead logging):** ext3/4, XFS, NTFS.
  1. Write the intended changes to the **journal** (a log area) + a commit record.
  2. Then write them to their real locations (**checkpoint**).
  3. After a crash: **replay** committed journal transactions, discard uncommitted ones ⇒ recovery in seconds.

  ext4 modes:
  | Mode | What goes in the journal | Guarantee |
  |---|---|---|
  | `journal` | metadata **and** data | strongest, slowest (data written twice) |
  | **`ordered`** (default) | metadata only; **data written to its final place before** the metadata commit | no garbage in files after a crash |
  | `writeback` | metadata only, no ordering | fastest; files may contain stale data after a crash |
- **Copy-on-write file systems** (ZFS, btrfs, **APFS** [APPLE]): never overwrite in place; write new blocks, then atomically switch the root pointer ⇒ always consistent. Snapshots and clones are almost free. APFS: clones (`cp` of a big file is instant), snapshots (Time Machine), encryption, optimised for flash.

### 2.7 Page cache, writeback and durability (killer topic)
```
write(fd, buf, n):
   copy buf → kernel page cache (page marked dirty)  →  RETURN to the app (µs)
   … later (dirty_expire ~30 s, or memory pressure, or fsync) …
   flusher threads write dirty pages → block layer → disk
```
- `write()` returning ≠ data on disk. A power loss can lose seconds of "written" data.
- **`fsync(fd)`:** blocks until the file's data **and metadata** are on stable storage (and the device cache is flushed). **`fdatasync`:** data + only the metadata needed to read it (skips mtime) ⇒ cheaper.
- **`O_DIRECT`:** bypass the page cache (DMA straight from user buffers; needs aligned buffers). Used by databases with their own cache. **`O_SYNC`/`O_DSYNC`:** every write behaves like write + fsync.
- **Durably replacing a file (classic question):**
  ```
  1. write the new content to "file.tmp"
  2. fsync(tmp_fd)            → data is on disk
  3. rename("file.tmp", "file")   → atomic on POSIX: readers see old OR new, never half
  4. fsync(directory_fd)      → the rename itself (a directory entry change) is durable
  ```
- **Read path:** `read()` checks the page cache; hit ⇒ memcpy (fast); miss ⇒ disk I/O + **readahead** (the kernel prefetches the next blocks on sequential access).
- [HFT] Logging on the hot path: never `write()`/`fsync()` from the trading thread. Write into a shared-memory ring buffer and let a background thread do the I/O.

### 2.8 VFS and "everything is a file"
- **VFS (Virtual File System):** a common interface (`open/read/write/stat`, inode/dentry/file objects) that every concrete FS (ext4, XFS, NFS, FAT, procfs) implements. User programs don't care which FS they're on.
- **Special file systems:** `/proc` (process and kernel info: `/proc/<pid>/maps`, `/proc/meminfo`), `/sys` (devices, drivers, tunables), `/dev` (device files: `/dev/sda` block device, `/dev/ttyS0` char device, `/dev/null`, `/dev/nvidia0`), `tmpfs` (RAM-backed, e.g. `/dev/shm`).
- **Mounting** attaches a file system's root at a directory in the single tree.

---

## 3. Worked Numericals

**N1. Maximum file size (inode).** Block = 4 KB, pointer = 4 B ⇒ pointers per block = 4096 / 4 = 1024. 12 direct, 1 single, 1 double, 1 triple indirect.
| Part | Blocks | Size |
|---|---|---|
| Direct | 12 | 48 KB |
| Single indirect | 1024 | 4 MB |
| Double indirect | 1024² ≈ 1.05 M | 4 GB |
| Triple indirect | 1024³ ≈ 1.07 G | 4 TB |
| **Total** | | **≈ 4 TB** (+ 4 GB + 4 MB + 48 KB) |
(The theoretical limit from pointers alone; real ext2/3 limits are lower because of other fields.)

**N2. Disk reads to access a byte.** Read byte offset 5,000,000 of `/a/b/f`, nothing cached, inodes and directories each fit in one block.
- Path: root inode (1) + root dir block (1) + inode a (1) + dir a (1) + inode b (1) + dir b (1) + inode f (1) = 7 reads.
- Which block? 5,000,000 / 4096 = block index **1220** (0-based). Direct covers 0–11; single indirect covers 12–1035; double indirect covers 1036 onwards ⇒ **double indirect**: read the double-indirect block (1) + the second-level block (1) + the data block (1) = 3 reads.
- Total = **10 disk reads**. (With 10 ms per HDD read ≈ 100 ms; that's why the caches matter.)

**N3. Bitmap size.** 1 TB disk, 4 KB blocks ⇒ 2⁴⁰ / 2¹² = 2²⁸ blocks ⇒ bitmap = 2²⁸ bits = 2²⁵ B = **32 MB**.

**N4. FAT size.** 32 GB SD card, 32 KB clusters, 4 B FAT entries: clusters = 32 GB / 32 KB = 2²⁰ ⇒ FAT = 2²⁰ × 4 B = **4 MB** (two copies are usually kept ⇒ 8 MB).

**Try it:** block 1 KB, pointer 4 B, 10 direct + 1 single + 1 double. Max file size?
<details><summary>Answer</summary>Pointers/block = 256. 10 KB + 256 KB + 256² KB = 10 + 256 + 65,536 KB = 65,802 KB ≈ <b>64.26 MB</b>.</details>

---

## 4. Interview Questions

### Basic
- **Q: What is an inode? Does it store the file name?** File metadata + block pointers. No name: names live in directory entries.
- **Q: Hard link vs soft link?** Hard link = another directory entry with the same inode number (every filename is one; the names are equal; data survives until link count 0). Soft link = its own inode containing a path, followed by name at lookup time (dangles if the name disappears; works across file systems and to directories). The §2.3.3 experiment shows it all. *Follow-up:* "You `rm` the target and recreate a file with the same name. What do the hard link and the symlink show?" Hard link: the old contents (old inode). Symlink: the new file.
- **Q: Why is `mv` of a 50 GB file instant within a disk but slow to another disk?** Within one FS it only adds a new directory entry and removes the old one (same inode, no data moved). Across file systems inode numbers don't carry over, so it must copy the data and then delete the original.
- **Q: Is a directory an inode? Where is a directory's name stored?** Yes: an inode of type directory whose data blocks hold (name → inode #) entries. Its name is stored in its **parent's** entries, not in its own inode. *Follow-up:* "Can you delete a read-only file you don't own?" Yes, if you have w + x on the directory (unless the directory has the sticky bit, like `/tmp`), because deletion edits the directory, not the file.
- **Q: What is a file descriptor?** A per-process integer index into the fd table, pointing to an open-file description (offset, flags) → inode.

### Intermediate
- **Q: After fork, parent and child write to the same fd. What happens to the offset?** Shared (same OFD) ⇒ writes interleave, no overwrite.
  *Follow-up:* "They run in parallel on two cores. How is the shared offset safe?" The OFD is one kernel object protected by `f_pos_lock`. Each write reserves [pos, pos+n) and advances the offset under the lock ⇒ whole writes interleave in some order, never overlap (POSIX; Linux ≥ 3.14). Separate `open()`s don't share an offset ⇒ they overwrite unless `O_APPEND`. Threads that want no shared-offset contention use `pwrite`.
- **Q: Why does `printf("hi"); fork();` print "hi" twice?** The stdio buffer is in user memory, which the child gets a copy of; both processes flush it at exit. Fix: `fflush(stdout)` before `fork`.
- **Q: I deleted a 50 GB log file but `df` shows no free space. Why?** A process still has it open; blocks are freed only when the link count = 0 and there are no open fds. Restart the process, or truncate via `/proc/<pid>/fd/N`.
- **Q: What is journaling? ext4 ordered mode?** §2.6.
- **Q: Contiguous vs linked vs indexed allocation?** §2.4.

### Advanced / killer follow-ups
- **Q: `write()` returned success; then the power failed and the data is gone. Why? How do you guarantee durability?** Page cache + writeback; use `fsync`/`fdatasync` (or `O_DSYNC`). For atomic replacement: tmp + fsync + rename + fsync dir.
- **Q: What happens when you run `cat file`?** `open` (path resolution via dentry/inode caches; permission check; fd allocated) → `read` loop (page cache hit, or disk read + readahead; copy to user) → `write` to fd 1 (terminal) → `close`.
- **Q [APPLE]: What does APFS do differently from HFS+/ext4?** Copy-on-write metadata, snapshots, instant clones, native encryption, space sharing between volumes, flash-optimised.
- **Q: Why do databases use `O_DIRECT`?** They manage their own buffer cache ⇒ avoid double caching and get control over write ordering/durability.
- **Q [HFT]: How do you log from a latency-critical thread?** Never block on I/O: write binary records into a lock-free ring buffer; a separate core formats and writes them (batching, async).

---

## 5. Common Traps
- Saying the file name is stored in the inode.
- Thinking `rm` immediately frees space.
- Thinking `close()` or `write()` guarantee the data is on disk.
- Forgetting the directory `fsync` after `rename` for durability.
- Mixing up fork-shared offsets (same OFD) with re-`open` (independent offsets).
- Thinking the parallel parent/child writes corrupt the offset. The kernel serialises them with the OFD's `f_pos_lock`; only the *order* is undefined.
- Thinking `fork` shares the fd **table**. The table is copied (closing in one doesn't close in the other); only the OFDs behind it are shared.
- Forgetting the indirect-block reads when counting disk accesses.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Three tables | fd table (per process) → open-file description (offset, flags) → inode |
| fork/dup | share the OFD ⇒ share the offset |
| Parallel writes, shared fd | fd table copied, OFD shared; `f_pos_lock` ⇒ each write reserves [pos, pos+n) atomically ⇒ interleave, no overwrite. Separate opens ⇒ overwrite unless O_APPEND; `pwrite` avoids the shared offset |
| Inode | metadata + block pointers; link count; **no name** |
| Directory | name → inode number |
| Directory = inode | type DIR; data blocks = entries; name lives in the **parent**; link count = 2 + subdirs; r = list, w = add/remove entries, x = traverse; root = #2 |
| Hard vs soft | hard = extra entry → same inode (every name is one; same FS, no dirs) / soft = own inode holding a path, followed by name (can dangle; recreate target ⇒ points to new file) |
| unlink | freed when link count 0 **and** not open |
| Allocation | contiguous, linked, FAT, indexed (12 direct + 1/2/3 indirect), extents |
| Max size (4 KB, 4 B ptrs) | 48 KB + 4 MB + 4 GB + 4 TB |
| Crash consistency | fsck → journaling (ext4 ordered default) → CoW (APFS, ZFS, btrfs) |
| Durability | write → page cache; fsync/fdatasync; tmp + fsync + rename + fsync(dir) |
| VFS | common interface; /proc, /sys, /dev, tmpfs |
