# 01 — Shell fundamentals with zsh

## What you’ll learn

The shell is not the operating system; it is a program that reads command text, constructs processes, and gives them an environment. You will learn zsh’s startup model, command lookup, configuration scope, and job control—the parts that matter when local shell habits meet a non-interactive Linux deployment.

You already know how to navigate and create files. Treat this module as the layer between your terminal and the kernel.

## Shell, terminal, and kernel

Terminal.app/iTerm provides a terminal device. `zsh` reads your input, expands it, finds a command, then usually asks the kernel to create a child process (`fork`/`exec`, conceptually). The child inherits a copy of the shell’s environment and open file descriptors. A shell built-in such as `cd` cannot be an ordinary child program: a child changing directory would not change its parent shell.

```
terminal emulator → pseudo-terminal (pty) ↔ zsh → kernel → child process
                                      stdin  stdout  stderr
                                       fd 0   fd 1    fd 2
```

`zsh` and `bash` share the Bourne-shell family and most scripting syntax. zsh deliberately has different defaults and more interactive features (notably powerful globbing and completion). A script must declare its interpreter: `#!/usr/bin/env bash` means “run under bash,” even if you launch it from zsh. Do not assume an interactive zsh feature works in `/bin/sh` or on a Linux server.

| Concern | macOS default | Typical Linux server |
|---|---|---|
| Login shell | zsh | bash, often; user configurable |
| `/bin/sh` | POSIX-oriented shell implementation, not a promise of bash/zsh | often `dash` (Debian/Ubuntu) or `bash` compatibility mode |
| User shell config | `~/.zshenv`, `~/.zprofile`, `~/.zshrc`, `~/.zlogin` | commonly `~/.bash_profile`, `~/.profile`, `~/.bashrc` |
| Deployment command shell | varies | frequently non-interactive `/bin/sh` or bash |

## Startup files: scope matters

zsh reads different files based on whether it is a login shell and/or interactive. `~/.zshenv` runs for *every* zsh, including scripts: keep it minimal. `~/.zprofile` is for login-session setup; `~/.zshrc` is for interactive conveniences (prompt, aliases, completion); `~/.zlogin` runs after login setup. Exact system-wide files vary.

Put an alias in `.zshrc`, not an API credential needed by an unattended deployment. Non-interactive remote commands may never read `.zshrc`. Put durable configuration in a service environment file, deployment secret store, or explicit script instead.

```zsh
# ~/.zshrc — interactive convenience only
alias k='kubectl'
mkcd() { mkdir -p -- "$1" && cd -- "$1" }
```

Expected behavior:

```console
$ mkcd scratch
$ pwd
/Users/you/scratch
```

An alias is text substituted by the interactive shell; it is not available to subprocesses. A function runs in the current shell and can use arguments (`$1`, `"$@"`) and return statuses. Use scripts for shared deployment behavior.

## Environment and command lookup

Shell variables exist in the shell. Exporting makes a variable part of the environment inherited by children:

```zsh
project_name=api                 # zsh only
export APP_ENV=staging           # child processes receive this
print -r -- "$APP_ENV"
env | grep '^APP_ENV='
```

Expected output:

```text
staging
APP_ENV=staging
```

`PATH` is an ordered list of directories used to find an unqualified executable. The first matching executable wins. It is not a security boundary; do not put `.` early in `PATH`, especially on servers.

```zsh
print -rl -- $path       # zsh's array view of PATH
command -v python3       # resolved command, builtin, function, or alias
whence -va ls            # zsh: all candidates
```

When something “works in my terminal but not in CI,” compare the environment rather than reinstalling packages: `env | sort`, `command -v tool`, working directory, user, and shell. CI/systemd commonly has a short, explicit `PATH`.

## Expansion and quoting

Before execution, the shell performs expansions. Quoting controls which occur. A reliable default is quote variable expansions unless you deliberately need splitting or globbing.

```zsh
file='two words.txt'
printf '<%s>\n' "$file"       # one argument
printf '<%s>\n' $file         # shell-dependent splitting behavior; avoid
printf '%s\n' -- *.md         # glob expands to matching pathnames
```

Use `--` before a path that could begin with `-`: `rm -- "$file"`. This tells many programs that later arguments are operands, not options.

## Jobs and terminal control

An interactive shell places a foreground job in control of its terminal. `Ctrl-C` sends `SIGINT` to that foreground process group; `Ctrl-Z` sends `SIGTSTP`, usually suspending it. `bg` resumes a stopped job in the background, `fg` returns it to foreground, and `jobs` lists shell-managed jobs.

```console
$ sleep 300
^Z
[1]+  Stopped                 sleep 300
$ bg %1
[1]+ sleep 300 &
$ jobs -l
[1]+ 12345 Running             sleep 300 &
$ kill %1
```

`cmd &` backgrounds a job but does not make it durable after logout. Use systemd (Module 07) for services, not `nohup` as a production supervisor.

## Why this matters for deployment

Deployments fail when they rely on an interactive profile, alias, accidental `PATH`, or zsh-specific syntax. Write scripts with an explicit interpreter, an explicit environment, quoted inputs, and observable exit statuses; let a service manager own long-running processes.

