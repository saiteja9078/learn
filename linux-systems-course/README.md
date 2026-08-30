# Linux Systems for Cloud Deployment — from a macOS Foundation

This course uses your M1 Mac as a Unix/POSIX laboratory. macOS is Darwin (an XNU kernel with BSD userland and APFS); a typical cloud server is Linux (a Linux kernel with GNU or other userland and usually ext4 or XFS). The command names sometimes overlap, but the transferable skill is understanding the kernel interfaces, processes, filesystems, permissions, and services underneath them.

You do **not** need a local VM. Run the macOS-labelled examples locally. For Linux-only exercises, use a small free-tier cloud instance only after you are comfortable with the material; see Module 07 for a low-risk workflow and shutdown reminders.

## Course map

1. [Shell fundamentals with zsh](01-shell-fundamentals-zsh.md) — shell startup, process environment, command lookup, functions, and jobs.
2. [Filesystems](02-filesystems.md) — names, inodes, descriptors, mounts, links, ownership, and APFS/ext4/XFS differences.
3. [Creating and writing a file](03-file-write-path.md) — from `open()` and `write()` to cache, kernel, storage, and durability.
4. [Memory management](04-memory-management.md) — virtual address spaces, RAM, paging, swap, allocation, and pressure.
5. [Process management](05-process-management.md) — lifecycle, process trees, signals, jobs, states, and supervision.
6. [System monitoring and introspection](06-monitoring-introspection.md) — evidence-driven diagnosis with macOS and Linux tools.
7. [Deployment-relevant Linux](07-linux-deployment-practice.md) — systemd, packages, server identities, networking, and macOS-to-Linux traps.

## Suggested path

Read Modules 01–05 in order and run the local examples in a disposable directory such as `~/tmp/linux-course`. Then use Module 06 to inspect your Mac. Finally, use Module 07 against a disposable Linux instance. Commands marked **Linux** should not be expected to work on macOS; commands marked **macOS** should not be copied to a production server without checking its documentation.

