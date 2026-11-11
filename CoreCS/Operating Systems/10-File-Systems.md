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

### 2.3 Hard links vs symbolic (soft) links
| | Hard link (`ln a b`) | Symbolic link (`ln -s a b`) |
|---|---|---|
| What it is | another directory entry pointing to the **same inode** | a small file whose content is a **path** |
| Link count | increases the inode's count | doesn't change the target's count |
| Delete original name | data stays (count > 0) | link becomes **dangling** |
| Across file systems | ❌ (inode numbers are per-FS) | ✅ |
| To directories | ❌ (would create cycles) | ✅ |
| Own inode? | no (shares one) | yes |
**Deleting a file = `unlink()`** removes one name and decrements the link count. Blocks are freed only when the **link count = 0 AND no process has it open**. That's why disk space isn't freed when you delete a huge log a process still writes to (`lsof | grep deleted`), and why "unlink a temp file right after opening it" is a trick for auto-cleanup.

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
- **Q: Hard link vs soft link?** §2.3 table.
- **Q: What is a file descriptor?** A per-process integer index into the fd table, pointing to an open-file description (offset, flags) → inode.

### Intermediate
- **Q: After fork, parent and child write to the same fd. What happens to the offset?** Shared (same OFD) ⇒ writes interleave, no overwrite.
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
- Forgetting the indirect-block reads when counting disk accesses.

---

## 6. Revision Summary
| Topic | Remember |
|---|---|
| Three tables | fd table (per process) → open-file description (offset, flags) → inode |
| fork/dup | share the OFD ⇒ share the offset |
| Inode | metadata + block pointers; link count; **no name** |
| Directory | name → inode number |
| Hard vs soft | same inode, same FS, no dirs / path file, can dangle |
| unlink | freed when link count 0 **and** not open |
| Allocation | contiguous, linked, FAT, indexed (12 direct + 1/2/3 indirect), extents |
| Max size (4 KB, 4 B ptrs) | 48 KB + 4 MB + 4 GB + 4 TB |
| Crash consistency | fsck → journaling (ext4 ordered default) → CoW (APFS, ZFS, btrfs) |
| Durability | write → page cache; fsync/fdatasync; tmp + fsync + rename + fsync(dir) |
| VFS | common interface; /proc, /sys, /dev, tmpfs |
