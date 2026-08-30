# 07 — Deployment-relevant Linux knowledge for a Mac user

## What you’ll learn

You will learn the Linux-specific operating conventions that matter most on a cloud server: package and service management, server identities and filesystem layout, networking, secure remote work, and the macOS assumptions most likely to fail.

## A server is a managed system, not your terminal

On a typical Linux distribution, `systemd` is PID 1 and manages services (“units”). It starts them at boot, restarts according to policy, keeps their logs, and can set environment, working directory, user, capabilities, and resource limits. Use it instead of an SSH command ending in `&`.

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My application
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/srv/myapp
EnvironmentFile=/etc/myapp/myapp.env
ExecStart=/srv/myapp/bin/myapp
Restart=on-failure
RestartSec=3
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

```console
# Linux, as a privileged administrator
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now myapp.service
$ systemctl status myapp.service
● myapp.service - My application
     Active: active (running) since ...
$ journalctl -u myapp.service -f
```

Keep secrets out of the unit file when it is broadly readable or committed. Restrict an environment file (for example root-owned, mode 0600) or use the cloud/secret-management mechanism chosen by your organization. Environment variables are convenient but can be exposed through process/service inspection; treat them with care.

## Packages: package manager versus language ecosystem

Homebrew installs developer tools under `/opt/homebrew` on Apple silicon and is normally operated as your user. Linux distribution packages install system components into FHS locations and are managed with privilege. Debian/Ubuntu use `apt`; RHEL-family systems use `dnf` (with `yum` often compatible); Alpine uses `apk`.

```console
# Debian/Ubuntu
$ sudo apt update
$ sudo apt install -y curl ca-certificates

# Fedora/RHEL family
$ sudo dnf install -y curl ca-certificates

# macOS — local developer environment, not server equivalence
$ brew install curl
```

Pin/track application dependencies using the relevant ecosystem (lock files, container digest, artifact repository) while using OS packages for OS-level libraries and tools. Do not blindly mix package sources, or run routine package updates during an incident without understanding release policy.

## Users, permissions, and layout

Log in with a named, non-root account; use `sudo` only for privileged actions. Run an application as a dedicated unprivileged account. Root can bypass ordinary mode-bit checks, so “it works with sudo” is not proof the service account can access its files.

Typical FHS conventions: `/etc` configuration, `/var/lib` durable application state, `/var/log` logs (though journald may be primary), `/run` runtime PID/socket state, `/tmp` temporary files, `/usr` installed programs/libraries, `/srv` service data, and `/home` user homes. Distribution conventions and container images vary; choose paths deliberately and make ownership explicit.

```console
# Linux
$ sudo useradd --system --home /nonexistent --shell /usr/sbin/nologin myapp
$ sudo install -d -o myapp -g myapp -m 0750 /srv/myapp /var/lib/myapp
$ namei -l /srv/myapp/config.yaml   # show each path component's access
```

`useradd` options differ across distributions—read `man useradd` on the target. A cloud image may already provide a preferred administrator account (for example `ubuntu`, `ec2-user`, or a provider-specific name).

## Network essentials

An application can be healthy but unreachable because it binds only to loopback, a firewall/security group blocks traffic, DNS points elsewhere, or a reverse proxy has no route. Inspect each layer.

```console
# Linux
$ ip addr                 # interfaces and addresses
$ ip route                # default route
$ ss -ltnp                # TCP listeners; 127.0.0.1 is loopback-only
$ curl -v http://127.0.0.1:8080/health
$ getent hosts example.com
$ dig +short example.com  # may require dnsutils/bind-utils package

# macOS counterparts
$ ifconfig
$ route -n get default
$ lsof -iTCP -sTCP:LISTEN -n -P
$ scutil --dns
```

For a public cloud workload, allow only required inbound traffic in the cloud security group/firewall and host firewall; use SSH keys, disable password/root SSH where appropriate, patch per policy, and avoid exposing databases or admin ports. Test from the actual client network, not only localhost.

## macOS-to-Linux gotchas

| macOS habit | Linux deployment reality |
|---|---|
| zsh interactive configuration supplies tools/variables | systemd/CI normally does not read your zsh files |
| BSD command flags work locally | GNU `sed`, `date`, `stat`, `ps`, `find` flags often differ; write portable scripts or declare GNU dependencies |
| case-insensitive APFS hides filename mistakes | Linux filesystems are normally case-sensitive: `Config.yaml` ≠ `config.yaml` |
| Homebrew path and packages exist | server `PATH`, package names, library paths, and versions are distribution-specific |
| `launchctl` manages local agents | systemd units manage most Linux server services |
| local port test proves it works | bind address, DNS, cloud firewall/security group, TLS proxy, and route all still matter |
| `sudo` fixes access | it can mask incorrect service user/group ownership |

## Free-tier cloud practice without a local VM

If you choose an AWS free-tier-eligible instance, create the smallest supported Linux instance in a region/account whose free-tier terms you have verified. Attach only the minimal disk, tag it, set a billing alarm/budget, restrict SSH in the security group to your current public IP, and stop **and terminate/delete associated storage/public IP resources** when the exercise is finished. Free-tier eligibility, products, and quotas change, so verify current provider documentation and your account’s billing dashboard before creating anything.

Suggested first exercise: create a dedicated user and `/srv/myapp`; write a tiny static HTTP service or use an existing web server; manage it with a systemd unit; inspect it with `systemctl`, `journalctl`, `ss`, `curl`, `ps`, `lsof`, `df`, and `free`; deliberately make a permission error; then fix it as the service user. Do not put real credentials or personal data on a learning instance.

## Why this matters for deployment

The Linux server differs from your Mac mostly in operational conventions and security boundaries, not in Unix fundamentals. Reliable deployment comes from explicit units, identities, paths, network policy, and observability—never from relying on an interactive shell state.

