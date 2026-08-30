# 04 — Memory management

## What you’ll learn

You will distinguish a process’s virtual address space from physical RAM and disk-backed storage, understand pages, allocation, swapping, and memory pressure, and interpret the main macOS/Linux differences without treating them as different fundamentals.

## Virtual memory is the contract

Each process sees a private, contiguous-looking **virtual address space**. Addresses in it are not direct RAM addresses. The MMU translates virtual pages to physical frames using page tables controlled by the kernel. Page permissions enforce isolation: one process normally cannot read another’s memory, and a data page is not executable unless its mapping allows it.

```
process A virtual pages        process B virtual pages
0x... code  ──────┐             0x... code  ──────┐
0x... heap  ───┐  │             0x... heap  ───┐  │
               ▼  ▼                              ▼  ▼
             page tables ── MMU ── physical RAM frames
                                       ▲       ▲
               shared file/cache pages ┘       └─ anonymous pages
```

The kernel establishes mappings lazily. An allocator may reserve virtual address range immediately but physical memory is often committed when code first touches a page. That touch can cause a **page fault**; this is normal when the page is not present yet. The kernel allocates a zero-filled page, maps a file-backed page from cache/storage, or handles copy-on-write, then restarts the instruction.

## Where process memory comes from

Programs obtain memory from the runtime allocator (`malloc`, language heap). The allocator requests larger regions from the kernel (historically `brk`, frequently `mmap`); it then subdivides them. Common virtual regions include executable code, shared libraries, heap, stacks, memory-mapped files, and kernel-provided shared pages.

Forking is economical because parent and child initially share physical pages marked copy-on-write (COW). On a write, the kernel gives the writer a private copy. This is why shells can create a process then execute another program efficiently.

File-backed mappings and the page cache connect memory and I/O: reading a file often means mapping or copying cached file pages, not necessarily reading the SSD again. “RAM used by cache” is typically reclaimable.

## RAM, swap, and pressure

**Physical RAM** is fast volatile memory. **Anonymous memory** is private process memory not backed by a normal file. Under pressure, kernels reclaim clean file cache first, then may write anonymous pages to swap/compressed memory or reclaim other pages. Swap extends survivability, not performance: paging heavily to SSD causes latency collapse (thrashing).

| Topic | macOS (Apple silicon) | Linux |
|---|---|---|
| Memory architecture | unified memory shared by CPU/GPU; fixed installed capacity | hardware/configuration varies; GPUs may be separate |
| Compression | memory compression is a major pressure response | optional zswap/zram; distribution/config dependent |
| Swap | dynamically managed files; inspect with `sysctl vm.swapusage` | swap partition/file, inspect `swapon --show` / `free -h` |
| Pressure signal | `memory_pressure`, Activity Monitor Memory Pressure graph | PSI (`/proc/pressure`), reclaim/OOM evidence, metrics |
| Out-of-memory response | pressure/compression and process termination when needed | kernel OOM killer chooses a victim when reclaim fails |

On a nearly full 256 GB Mac, storage pressure also constrains swap and snapshots. Leave meaningful free disk space; deleting active swap or system files is not a remedy. Reduce large local artifacts, caches, downloads, or use remote build systems deliberately.

## Inspect it safely

```console
# macOS
$ vm_stat
Mach Virtual Memory Statistics: (page size of 16384 bytes)
Pages free:                               ...
Pages active:                             ...
$ memory_pressure
System-wide memory free percentage: ...
$ sysctl vm.swapusage
vm.swapusage: total = ...M  used = ...M  free = ...M

# Linux
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       1.2Gi       0.4Gi       ...       2.2Gi       2.3Gi
Swap:          1.0Gi          0B       1.0Gi
```

On Linux, focus on `available`, not merely `free`: it estimates memory usable without swapping by reclaiming caches. `RSS` is resident physical memory currently associated with a process; `VSZ`/virtual size can be far larger and is not proof of RAM use. Shared pages make summing per-process RSS misleading.

## Why this matters for deployment

Right-sizing services means measuring memory pressure, RSS trends, cache, and OOM events—not reading one “free memory” number. Limits in containers and systemd can cause an OOM kill even if the host looks healthy, so budget memory explicitly and expose metrics before production traffic does the teaching.

