# 02 — Filesystems

## What you’ll learn

You will separate a pathname from the file object it reaches, understand inode-like metadata, file descriptors and mounts, and reason about permissions and links across macOS APFS and Linux ext4/XFS.

## Names, objects, and descriptors

A directory is a mapping from a filename to a filesystem object identifier. On traditional Unix filesystems that object is an **inode**, containing metadata (owner, mode, timestamps, size, block references) but usually not the filename. Multiple directory entries may point to one inode: that is a hard link. An open process uses a small integer **file descriptor** (FD), which refers to an open-file description holding offset and status flags.

```
/srv/app/config → directory entry → inode / file object → data blocks
process fd 3   → open-file description (offset, flags) ─┘
```

This explains two useful facts: renaming changes a directory entry, not file contents; and an unlinked file can still consume space while a process has it open. A server log “deleted” during rotation may remain allocated until the writer closes/reopens it.

```console
$ printf 'hello\n' > original
$ ln original hard
$ ln -s original symbolic
$ ls -li original hard symbolic
123456 -rw-r--r-- 2 you staff 6 ... hard
123456 -rw-r--r-- 2 you staff 6 ... original
123789 lrwxr-xr-x 1 you staff 8 ... symbolic -> original
```

The equal number and link count `2` identify hard links. A symbolic link has its own object and stores a path; it can cross filesystems and can dangle. A hard link normally cannot cross a filesystem and directories are restricted to preserve a tree.

## Mounts and the single tree

Unix presents one pathname tree rooted at `/`. A mount attaches another filesystem at a directory, hiding any underlying entries there while mounted. `/Volumes/...` is conventional on macOS; Linux commonly mounts data at `/mnt`, `/media`, or application-specific paths such as `/srv`. The kernel resolves a pathname component by component, crossing mount boundaries when necessary.

```console
$ df -h .
Filesystem      Size   Used  Avail Capacity  Mounted on
/dev/disk3s1s1  460Gi  ...   ...   ...       /
```

On Linux, `findmnt` shows a more explicit mount tree; `mount` output is useful but noisier. Container bind mounts and volumes make this model essential: `/data` inside a container can refer to host storage, not data baked into the image.

## Permissions and ownership

Every object has an owning UID, GID, and mode bits: read (`r=4`), write (`w=2`), execute (`x=1`) for owner, group, and others. The kernel evaluates credentials (effective UID/GID plus supplementary groups) rather than user names; names are presentation from local files or identity services.

For regular files, `r` allows reading and `w` changing contents. For directories, `r` lists names, `w` creates/removes/renames entries (subject to directory rules), and `x` means *search/traverse*—required to access a known child. Thus a directory can be listable but not traversable, or traversable but not listable.

```console
$ mkdir private && chmod 700 private
$ stat -f '%Sp %Su %Sg %N' private       # macOS
drwx------ you staff private
# Linux equivalent: stat -c '%A %U %G %n' private
```

`chmod 640 file` means owner `rw-` (6), group `r--` (4), others `---` (0). `umask` removes permissions from defaults; `umask 022` commonly yields `644` files and `755` directories. It does not add permissions.

Additional controls exist: POSIX ACLs on both platforms, macOS extended attributes and ACLs, and Linux capabilities, SELinux/AppArmor on many server distributions. Traditional mode bits are necessary but are not the complete access-control story.

| Topic | macOS | Linux |
|---|---|---|
| Default local filesystem | APFS: copy-on-write, snapshots, volumes, space sharing | ext4 often; XFS common on enterprise/cloud systems |
| Inode model | POSIX-visible inode numbers, APFS internals differ | ext4/XFS use native inode metadata structures |
| Case behavior | APFS usually case-insensitive by default, can be sensitive | ext4/XFS usually case-sensitive |
| Metadata tooling | BSD `stat -f`, `ls -l@` for extended attrs | GNU `stat -c`, `getfacl`, `ls -Z` where SELinux applies |
| Disk reporting | APFS volumes share an APFS container | each mounted filesystem has independently managed free space |

APFS’s copy-on-write and snapshots are not a license to ignore backups or durability. ext4 usually journals metadata; XFS is a high-scale journaling filesystem. Their allocation and recovery details differ, but applications should use normal atomic-update patterns: write a new file in the same directory/filesystem, `fsync` if required, then `rename` it into place.

## File descriptors in practice

Standard input/output/error are FDs 0/1/2. Redirection rewires descriptors before a process begins:

```console
$ sh -c 'echo normal; echo problem >&2' >out.txt 2>err.txt
$ cat out.txt; cat err.txt
normal
problem
```

Use `lsof -p PID` to inspect a process’s open files/sockets. On Linux, `/proc/PID/fd` exposes descriptor links; macOS has no identical `/proc` interface.

## Why this matters for deployment

Correct ownership and directory traversal permissions prevent both outages and over-broad access. Understanding mounts prevents data-loss surprises with containers, and FD/inode reasoning lets you diagnose stubborn “disk full” incidents after a log file was deleted.

