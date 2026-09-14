# OpenBSD — Administration Cheatsheet

Covers package management, the `rc.d`/`rcctl` init system, service/daemon management, users, filesystems, and networking. OpenBSD does not use systemd — its init system is BSD-style `/etc/rc` plus the `rc.d(8)` framework, controlled via `rcctl(8)`.

---

## 1. System Basics

```sh
uname -a                  # kernel/version info
sysctl hw.model            # CPU model
sysctl hw.ncpu             # CPU core count
dmesg                      # boot log
doas <cmd>                 # run command as root (sudo equivalent, configured in /etc/doas.conf)
```

`doas` is the default privilege-escalation tool (sudo is available via packages but not installed by default). Minimal `/etc/doas.conf`:

```
permit persist keepenv :wheel
```

---

## 2. Package Management (`pkg_add` family)

```sh
pkg_add <pkg>               # install a package
pkg_add -u                  # update all installed packages
pkg_delete <pkg>             # remove a package
pkg_info                     # list installed packages
pkg_info -Q <string>          # search available packages by name
pkg_info <pkg>                # show details/description for a package
```

Set the mirror once via `/etc/installurl` (installer usually does this automatically):

```sh
echo "https://cdn.openbsd.org/pub/OpenBSD" | doas tee /etc/installurl
```

### Ports tree (building from source)

```sh
cd /usr/ports/www/nginx && doas make install clean
```

Most administrators use binary packages (`pkg_add`) and only build from ports when a package needs custom options.

---

## 3. Init System: `rc.d` and `rcctl`

OpenBSD's init is a simple shell-script-driven system (`/etc/rc`, `/etc/rc.d/*`), managed day-to-day through the `rcctl(8)` wrapper — this is the equivalent of `systemctl` on Linux.

```sh
rcctl ls all                 # list all known services and their status
rcctl ls on                  # list services enabled at boot
rcctl ls started             # list currently running services
rcctl status httpd           # show status of one service

rcctl enable httpd           # enable service at boot (writes to /etc/rc.conf.local)
rcctl disable httpd          # disable service at boot
rcctl start httpd            # start now
rcctl stop httpd             # stop now
rcctl restart httpd          # restart
rcctl reload httpd           # reload config (if the daemon supports SIGHUP-style reload)

rcctl set httpd flags "-syslog"   # set custom startup flags for a service
rcctl get httpd status            # query stored config value
```

- Service enable/disable state and flags live in **`/etc/rc.conf.local`** (never edit the shipped `/etc/rc.conf` directly — it's overwritten on upgrades).
- `pkg_scripts` in `rc.conf.local` lists third-party daemons started at boot, in order.
- To see what a package registered as its rc.d script: `rcctl ls all | grep <name>`.

### Boot sequence overview

1. Kernel boots, `/etc/rc` runs.
2. Base system services from `rc.conf`/`rc.conf.local` start (in the order `rc.d` determines from dependency-free simple sequencing — OpenBSD's rc.d has no complex dependency graph like systemd units).
3. `pkg_scripts` daemons start.
4. `/etc/rc.securelevel` and `/etc/rc.local` run for any final custom startup commands.

---

## 4. Common Base-System Daemons

| Daemon | Purpose | Config |
|---|---|---|
| `sshd` | SSH server | `/etc/ssh/sshd_config` |
| `httpd` | Native lightweight web server | `/etc/httpd.conf` |
| `relayd` | Load balancer / reverse proxy / TLS termination | `/etc/relayd.conf` |
| `smtpd` | Mail transfer agent | `/etc/mail/smtpd.conf` |
| `ntpd` | Time sync | `/etc/ntpd.conf` |
| `pf` (via `pfctl`) | Firewall/NAT | `/etc/pf.conf` |
| `dhcpd` / `dhcleased` | DHCP server/client | `/etc/dhcpd.conf` |
| `unbound` | Recursive DNS resolver (base system since 6.x) | `/var/unbound/etc/unbound.conf` |
| `slaacd` | IPv6 SLAAC client | n/a (automatic) |

Check/enable a base daemon:

```sh
rcctl enable sshd pf ntpd
rcctl start sshd pf ntpd
```

---

## 5. Packet Filter (`pf`)

```sh
pfctl -e                    # enable pf
pfctl -d                    # disable pf
pfctl -f /etc/pf.conf        # load/reload ruleset
pfctl -sr                    # show active rules
pfctl -ss                    # show current state table
pfctl -si                    # show filter statistics
pfctl -sn                    # show nat rules
```

pf is enabled by default at boot via `rcctl enable pf` and reads `/etc/pf.conf`. Always test rule syntax before reload:

```sh
pfctl -nf /etc/pf.conf       # dry-run parse check, no load
```

---

## 6. Users & Groups

```sh
useradd -m -G wheel <user>    # add user, create home dir, add to wheel group
usermod -G wheel <user>       # modify group membership
userdel -r <user>             # delete user and home dir
passwd <user>                 # set/change password

groupadd <group>
groupdel <group>
```

`wheel` group membership is what `doas.conf`'s `:wheel` rule authorizes for privilege escalation.

---

## 7. Filesystem & Storage

```sh
df -h                        # disk usage
mount                        # show mounted filesystems
disklabel sd0                # view/edit disk partition layout
newfs /dev/rsd0a             # create FFS filesystem
fsck -y /dev/sd0a             # check/repair filesystem

bioctl -c C -l /dev/sd0a softraid0   # create a softraid crypto (encrypted) volume
```

`/etc/fstab` controls mounts at boot, same format as other BSD/Unix systems.

---

## 8. Logging

```sh
tail -f /var/log/messages     # main system log
tail -f /var/log/daemon       # daemon-specific log
tail -f /var/log/authlog       # authentication log
newsyslog                      # manually trigger log rotation (normally cron-driven)
```

Logging is handled by `syslogd`, configured via `/etc/syslog.conf`; rotation via `newsyslog.conf`.

---

## 9. Patching & Upgrades

```sh
syspatch                     # apply all available binary security patches for the current release
syspatch -l                   # list installed patches
syspatch -r                   # rollback the most recently applied patch

sysupgrade                    # upgrade to the next release (downloads, verifies, reboots into installer)
```

`syspatch` handles in-place security fixes within a release; `sysupgrade` handles moving between major releases (e.g., 7.4 → 7.5).

---

## 10. Quick Reference: Linux → OpenBSD Equivalents

| Linux | OpenBSD |
|---|---|
| `systemctl enable/start/stop <svc>` | `rcctl enable/start/stop <svc>` |
| `sudo` | `doas` |
| `apt`/`dnf`/`pacman` | `pkg_add` / `pkg_delete` |
| `iptables`/`nftables` | `pf` (`pfctl`) |
| `/etc/systemd/system/*.service` | `/etc/rc.d/*` scripts |
| `journalctl` | `/var/log/messages` + `syslogd` |
| SELinux/AppArmor | `pledge(2)` / `unveil(2)` (per-process, not policy-file based) |
