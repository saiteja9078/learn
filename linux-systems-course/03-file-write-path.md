# 03 — What happens when a file is created and written

## What you’ll learn

You will follow a write from application code through a system call, VFS and page cache to durable storage, and distinguish “the call succeeded” from “the bytes survived a power loss.”

## The path, step by step

An application might call `open("report", O_CREAT|O_WRONLY, 0644)`, `write(fd, bytes, length)`, then `close(fd)`. Language runtimes may buffer first: Python, Node, and C stdio can hold bytes in user-space memory until they flush to the kernel. Once `write()` occurs, the kernel path is broadly:

```
application buffer
  │ flush / write(fd, bytes)
  ▼
system-call boundary ── kernel validates FD, permissions, arguments
  ▼
VFS: resolves pathname / chooses mounted filesystem implementation
  ▼
filesystem: creates directory entry + inode/file metadata; maps file offsets
  ▼
page cache (RAM): copied/attached pages become dirty
  ▼ asynchronously
writeback: filesystem + block layer + storage driver
  ▼
SSD controller cache / flash translation layer → NAND flash
```

**VFS** (Virtual Filesystem Switch) is an abstraction: application calls such as `open`, `read`, `write`, and `stat` work across APFS, ext4, XFS, NFS, and more. The VFS dispatches into the mounted filesystem’s implementation. The precise internal names differ between XNU/macOS and Linux, but the architectural role is the same.

## Cache, RAM, and virtual memory

The page cache is kernel-managed physical RAM used to cache file contents. On a write, the kernel commonly marks cached pages **dirty** and returns before those pages are written to the device. This is good: it batches writes, coalesces small changes, and makes a subsequent read fast. It also means free RAM being used for cache is healthy—it can be reclaimed if applications need it.

The application’s buffer is in its **virtual address space**. Virtual pages map to physical RAM pages; copying to the kernel’s cache crosses protection boundaries. Some high-performance paths reduce copies, but the correctness model remains: the kernel controls cached file state and later writeback.

| Question | Ordinary `write()` | `fsync(fd)` / equivalent |
|---|---|---|
| Are bytes accepted by the kernel? | normally yes on success | yes |
| Are dirty pages necessarily on persistent media? | no | requested to be made durable before success |
| Is the whole update transaction durable? | not necessarily | depends on syncing required file and directory metadata |
| Performance | high through batching | slower; may wait for device/filesystem flush |

`close()` does **not** universally mean a power-safe commit. `fsync()` asks the OS to flush a file’s data and relevant metadata; an atomic-replace sequence also needs the parent directory synced when crash consistency matters. Filesystems, device caches, and hardware write barriers all participate. Databases implement careful write-ahead logging and fsync policies for this reason—do not casually replace them with “write a JSON file.”

## Creation is metadata plus data

Creating a file changes directory metadata and allocates/initializes an inode-like object. Writing adds or maps data blocks/extents and updates size/timestamps. Journaling filesystems order or journal metadata so crash recovery can restore filesystem consistency; that does not necessarily make the last application-level record present. APFS uses copy-on-write metadata; ext4/XFS have different journaling/ordering designs. All have documented edge cases and mount options.

An atomic configuration update commonly looks like this:

```text
1. create temp file beside target (same filesystem)
2. write complete new contents
3. fsync temp file if durability is required
4. rename temp → target (atomic name replacement on one filesystem)
5. fsync parent directory if crash durability is required
```

Never use a cross-filesystem rename expecting atomic replacement; it becomes copy-plus-delete behavior in higher-level tools.

## Observe, don’t over-interpret

```console
$ printf 'one\n' > demo.txt
$ ls -l demo.txt
-rw-r--r--  1 you  staff  4 Aug 30 12:00 demo.txt
$ sync                         # request system-wide pending writes
```

`sync` is a diagnostic/administrative tool, not a replacement for an application’s targeted durability design. macOS offers `fs_usage` (usually requires elevated privileges) to observe filesystem activity. Linux offers `strace -e trace=%file,write program` to see system calls and tools such as `iostat` for device activity. Timing of writeback is intentionally implementation-dependent.

## Why this matters for deployment

This model explains why a successful request can still lose recent data after a crash, why “used memory” is not automatically a leak, and why robust deployments use database transactions, atomic config writes, durable volumes, backups, and graceful shutdown rather than assuming `write()` immediately reaches an SSD.

