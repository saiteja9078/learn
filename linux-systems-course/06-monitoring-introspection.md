# 06 — System monitoring and introspection

## What you’ll learn

You will use a small, portable diagnostic toolkit to identify whether a deployment problem is CPU, memory, disk capacity, I/O, descriptors, network, or service state—and read the numbers in context.

## Begin with a question and a time window

“The server is slow” is not yet a diagnosis. Establish: which host/container, when it began, which service/request, and whether the symptom is latency, errors, saturation, or data loss. Sample repeatedly; a single `top` screen is a snapshot, not a causal explanation.

## Processes, CPU, and memory

`top` is interactive on both platforms, but its flags and columns differ. `htop` is optional and friendlier on Linux. `ps` is scriptable and useful for sorted snapshots.

```console
# macOS
$ top -l 1 -o cpu | head -20
$ ps -axo pid,ppid,%cpu,%mem,rss,stat,command | sort -nrk3 | head

# Linux
$ top
$ ps -eo pid,ppid,%cpu,%mem,rss,stat,etime,cmd --sort=-%cpu | head
$ uptime
 12:00:00 up 2 days,  3:00,  1 user,  load average: 0.34, 0.28, 0.20
```

Load average is the average count of runnable work (and, on Linux, often tasks waiting in uninterruptible I/O) over 1, 5, and 15 minutes. Compare it with CPU count and symptoms; it is not a CPU percentage. `%CPU` can exceed 100 on a multi-core machine depending on tool semantics.

For memory, use macOS `memory_pressure`, `vm_stat`, and Activity Monitor; use Linux `free -h`, `vmstat 1`, and cgroup/container metrics. A growing RSS plus pressure/OOM events is more meaningful than cache consumption.

## Filesystems, I/O, and deleted-but-open files

```console
$ df -h                 # capacity by mounted filesystem
$ du -sh .              # apparent directory usage; may be slow
$ du -sh * 2>/dev/null  # locate large direct children (shell expansion)

# Linux: inode exhaustion is separate from byte capacity
$ df -ih
$ lsof +L1              # open files with link count < 1 (deleted)
```

`df` reports filesystem allocation; `du` walks names visible from a directory. They can disagree when data is open-but-unlinked, hidden under a mount point, sparse, deduplicated, snapshot-retained, or excluded by permissions. On macOS, use `lsof | grep '(deleted)'` carefully as an investigative starting point; output conventions differ.

`vmstat 1` gives repeating CPU, runnable, memory, paging, and I/O indicators; labels differ by OS. Linux `iostat -xz 1` (package may be `sysstat`) helps identify a saturated device. Correlate high latency/queue/utilization with application latency; high throughput alone can be healthy.

## Open files, listeners, and network evidence

```console
# both, with flag/output differences
$ lsof -iTCP -sTCP:LISTEN -n -P

# macOS
$ netstat -anv | head
$ log show --last 15m --predicate 'process == "myapp"'

# Linux
$ ss -ltnp
$ journalctl -u myapp.service --since '15 minutes ago'
$ journalctl -p warning..alert --since today
```

`lsof` answers “which process owns this file/socket?” It is indispensable for port conflicts and deleted log files. `ss` is Linux’s modern socket view; `netstat` remains present on some systems but may require an extra package. Use `-n` to avoid slow DNS/service-name resolution while debugging.

## Logs: service context matters

macOS’s unified logging can be queried with `log show` (historical) and streamed with `log stream`; Console is its GUI. Most systemd Linux hosts send service stdout/stderr and service-manager events to the journal, queried with `journalctl`. Other Linux services write files under `/var/log`, and container platforms have their own log drivers.

| Need | macOS | Linux/systemd |
|---|---|---|
| Current process/resource view | `top`, Activity Monitor, `ps` | `top`/`htop`, `ps`, `systemd-cgtop` |
| Memory | `memory_pressure`, `vm_stat` | `free`, `vmstat`, `/proc/meminfo` |
| Logs | Console, `log show`, `log stream` | `journalctl`, `/var/log/*` |
| Service health | `launchctl print ...` | `systemctl status ...` |
| Open ports | `lsof -i`, `netstat` | `ss -ltnp`, `lsof -i` |

## A compact incident sequence

1. Confirm target and current errors/latency; preserve timestamps in UTC where possible.
2. Check service state and recent logs.
3. Check capacity (`df`), pressure, CPU/load, and I/O over a short interval.
4. Identify owning PID/cgroup and inspect open files, sockets, command line, limits.
5. Form a hypothesis, make the smallest safe change, then measure again.

Avoid reflexively restarting: it destroys useful evidence and can turn a capacity or dependency issue into a repeating outage.

## Why this matters for deployment

Deployment work is operations work once traffic arrives. A structured evidence loop lets you distinguish a bad release from exhausted disk, a missing listener, an OOM kill, or a dependency timeout—and makes rollback or remediation deliberate.

