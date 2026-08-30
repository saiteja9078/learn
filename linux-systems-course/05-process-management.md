# 05 — Process management

## What you’ll learn

You will learn how processes are born, replaced, observed, signalled, and reaped; how shell jobs relate to processes; and why production services need a supervisor.

## Lifecycle and ancestry

A process is a running program plus credentials, virtual memory, open descriptors, signal dispositions, and kernel scheduling state. A parent creates a child; on Unix this is classically `fork()`. The child can then use `execve()` to replace its program image while keeping selected inherited state. `exec` does not create a PID; it transforms the existing process.

```
launchd (macOS PID 1) / systemd (Linux PID 1)
                │
             shell / service manager
                │ fork
                ▼
              child ── execve ──► application process
                │ exit(status)
                ▼
              zombie until parent wait()s
```

After a child exits, its parent receives/observes status with `wait()`. Until then it is a **zombie**: no running memory, but a process-table record retains PID and exit status. If the parent dies, children are reparented to PID 1 (or an appropriate subreaper), which should reap them. Persistent zombies identify a faulty parent, not a process to “kill.”

## PIDs, process groups, sessions, and jobs

PIDs identify processes temporarily and can be reused; never use an old PID as a durable identity. A process group groups related processes, especially a pipeline. A session owns a controlling terminal. Your interactive shell uses foreground process groups to route terminal-generated signals.

Shell **jobs** are the shell’s records for one or more processes. A background job is not a daemon and may receive `SIGHUP` on terminal/session loss. `disown` changes a shell’s handling, but service managers are the reliable server mechanism.

```console
$ sleep 120 &
[1] 4242
$ jobs -l
[1]+ 4242 Running                 sleep 120 &
$ ps -o pid,ppid,pgid,stat,command -p 4242
  PID  PPID  PGID STAT COMMAND
 4242  4100  4242 S    sleep 120
```

Column availability/order differs slightly on BSD/macOS and GNU/Linux. Ask `ps` documentation on the target (`man ps`).

## States and scheduler view

Common `ps` state letters are `R` runnable/running, `S` interruptible sleep, `D` uninterruptible I/O sleep (Linux), `T` stopped, and `Z` zombie. A process shown as `S` is usually waiting efficiently, not necessarily broken. `D` that persists alongside storage/network symptoms can be important; do not treat state letters alone as diagnosis.

Threads share one process address space but have separate execution contexts. Linux tools may expose per-thread IDs; macOS and Linux schedule threads, not an abstract monolithic process.

## Signals: requests, not magic

Signals are asynchronous notifications. A process may catch, ignore, or handle many signals; `SIGKILL` and `SIGSTOP` cannot be handled or ignored. Use the least forceful signal first:

```console
$ kill -TERM 4242     # request graceful shutdown; default signal
$ kill -HUP 4242      # conventionally reload config for some daemons
$ kill -KILL 4242     # immediate kernel termination; no cleanup
```

`SIGTERM` is a request to exit cleanly: close listeners, finish/abort work according to policy, flush state. Give it a timeout, then use `SIGKILL` only as escalation. `Ctrl-C` normally sends `SIGINT` to the foreground group. `kill` sends a signal—it does not necessarily “kill.”

| Concern | macOS | Linux |
|---|---|---|
| PID 1 / service owner | `launchd` | `systemd` on most distributions |
| Process tree | `ps -ax -o ...`, `pstree` if installed | `ps -ef --forest`, `pstree` commonly available |
| Service unit management | `launchctl` | `systemctl` |
| Kernel process view | no Linux `/proc` equivalent | rich `/proc/PID/*` interface |

## Practical investigation

```console
# macOS
$ ps -ax -o pid,ppid,stat,%cpu,%mem,command | head
$ lsof -p 4242

# Linux
$ ps -eo pid,ppid,stat,%cpu,%mem,etime,cmd --forest | head -20
$ systemctl status myapp.service
$ cat /proc/4242/status
```

When a deployment starts a process, record its unit/service identity and logs, rather than saving only a PID in a file. PIDs are useful evidence, not a management API.

## Why this matters for deployment

Graceful rollout, log rotation, health checks, and incident response all rely on signals and parent-child ownership. A supervisor such as systemd restarts an expected service, captures logs, applies resource limits, and reaps children—things an SSH session and `&` do not reliably provide.

