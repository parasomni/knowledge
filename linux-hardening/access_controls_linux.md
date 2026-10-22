# Access Controls: Linux

> Companion note: [Access Controls: Windows](../windows-hardening/access_controls_windows.md)

## Overview

| Layer | Mechanism | Purpose |
|---|---|---|
| **DAC** | Unix permissions (UID/GID), POSIX ACLs (`setfacl`) | Identity-based "who may do what" |
| **Privileges** | POSIX capabilities, `sudo`, PAM | Split root into fine-grained rights |
| **MAC** | SELinux / AppArmor (LSM framework) | Policy-enforced, mandatory access |
| **Syscall filter** | Seccomp / seccomp-bpf | Restrict *which* syscalls a process may issue |
| **Self-sandbox** | Landlock | Unprivileged process confines itself |
| **Isolation** | Namespaces + cgroups | Limit *visibility* and resources (containers) |
| **Integrity** | IMA/EVM, `lockdown`, `yama` | Runtime tamper detection, kernel lockdown |

All layers complement each other — no single mechanism is sufficient on its own.

---

## 1. DAC – Unix Permissions & ACLs

- Classic `rwx` permissions based on UID/GID (owner / group / other)
- `umask` sets the default permission mask for newly created files
- **Special bits** change execution identity and directory semantics:

| Bit | Octal | Effect |
|---|---|---|
| setuid | `4000` | Executable runs as the file *owner* (e.g. `passwd`) |
| setgid | `2000` | Executable runs as the file *group*; on a dir, new files inherit the group |
| sticky | `1000` | In a dir, only the file owner may delete their files (e.g. `/tmp`) |

- Extendable via **POSIX ACLs** for per-user/per-group grants beyond the three classes:

```
    setfacl -m u:alice:rwx file      # grant alice rwx
    setfacl -m g:devs:r-x dir        # grant group devs r-x
    getfacl file                     # inspect ACLs
    setfacl -b file                  # strip all ACLs
```

- A trailing `+` in `ls -l` output (`-rw-rw-r--+`) marks a file carrying an ACL.

> **Audit the setuid surface** — every setuid-root binary is a privilege-escalation candidate:
> `find / -perm -4000 -type f 2>/dev/null`

---

## 2. Privileges – Capabilities, sudo, PAM

### POSIX Capabilities

The monolithic root privilege is split into ~40 discrete capabilities, so a service
can get exactly one power instead of full root.

| Capability | Grants |
|---|---|
| `CAP_NET_BIND_SERVICE` | Bind to ports < 1024 |
| `CAP_NET_ADMIN` | Configure interfaces, routing, firewall |
| `CAP_SYS_ADMIN` | Broad admin ops (overloaded — treat as near-root) |
| `CAP_DAC_OVERRIDE` | Bypass file read/write/execute permission checks |
| `CAP_SETUID` / `CAP_SETGID` | Change process UID/GID |

```
    getcap /usr/bin/ping             # inspect file capabilities
    setcap cap_net_bind_service=+ep /usr/local/bin/myserver
    getpcaps <pid>                   # capabilities of a running process
```

### sudo

- Least-privilege command delegation via `/etc/sudoers` (edit with `visudo`)
- Prefer explicit command allow-lists over blanket `ALL=(ALL) ALL`
- Avoid `NOPASSWD` except for well-scoped automation accounts

### PAM (Pluggable Authentication Modules)

- Pluggable auth/account/session/password stack under `/etc/pam.d/`
- Enforces password quality (`pam_pwquality`), lockout (`pam_faillock`),
  and access limits (`pam_limits`, `pam_access`)

---

## 3. MAC – Linux Security Modules (LSM)

Access controls enforced in the **LSM framework** directly inside the kernel —
inline on every syscall, *before* the access takes place.

### SELinux
- Labels on subjects (processes) and objects (files, ports, …)
- Policy defines allowed transitions (**Type Enforcement**)
- Modes: `disabled` → `permissive` (log only) → `enforcing`
- Used on RHEL / Fedora / Android

### AppArmor
- **Path-based** profiles per process (confinement by file path)
- Defines allowed file accesses, capabilities, network operations
- Modes: `complain` (log only) → `enforce`
- Easier to configure than SELinux; default on Debian / Ubuntu / SUSE

### Other LSMs

| LSM | Role |
|---|---|
| `landlock` | Unprivileged processes sandbox *themselves* by restricting their own filesystem access — limits an app's own blast radius |
| `lockdown` | Restricts what even root can do under Secure Boot (no unsigned modules, no `/dev/mem`) — hardens the kernel against privileged attackers |
| `yama` | Controls `ptrace` scope; typically only a parent may ptrace its children, blocking process-injection attacks |
| `integrity` | Umbrella for IMA/EVM — cryptographically measures/verifies files at runtime (a runtime tripwire for binary tampering) |
| `bpf` | Restricts who may load eBPF programs (eBPF runs code in the kernel) — keeps it to privileged users only |

> **LSM order matters.** The kernel processes the `lsm=` list left to right; the
> first LSM becomes "primary" for certain exclusive hooks. AppArmor must be
> listed explicitly in `lsm=` or the kernel won't activate it even when compiled in.

---

## 4. Seccomp – Syscall Filtering

- Whitelist/blacklist of syscalls per process (seccomp-bpf)
- Restricts *which system calls* a process may make at all — shrinks the kernel attack surface
- Used by browsers, `systemd` units, and container runtimes
- A syscall outside the filter is killed (`SIGKILL`) or returns `EPERM`

---

## 5. Isolation – Namespaces & cgroups

Namespaces are **not** an access-control mechanism — they are an **isolation**
mechanism. They limit *visibility*; LSM modules control *access*.

| Namespace | What is isolated? |
|---|---|
| `pid` | Process IDs |
| `net` | Network stack (interfaces, routing, firewall) |
| `mnt` | Filesystem mounts |
| `uts` | Hostname, domain name |
| `ipc` | IPC objects (shared memory, semaphores) |
| `user` | UID/GID mapping (root inside, unprivileged outside) |
| `cgroup` | cgroup root view |
| `time` | System clocks |

- **cgroups** limit resources (CPU, RAM, I/O, PIDs) — defense against resource-exhaustion DoS
- Namespaces + cgroups form the core of **containers** (Docker, LXC, Podman)

> **Isolation ≠ Access Control:** Namespaces limit *visibility*, LSM modules
> (SELinux/AppArmor) control *access*.

---

## 6. Wazuh syscheck – Detection, Not Enforcement

`syscheck` is the **File Integrity Monitoring (FIM)** module of Wazuh:

- Detects file changes *after the fact* (hash, inode, permission comparison)
- Lives in the SIEM, not in the kernel path
- **Not** an enforcement mechanism — it blocks nothing, it *reports*

Actual enforcement is handled by the kernel (LSM, DAC) and process level
(Seccomp, Capabilities). FIM/SIEM is the detective layer on top.

---

## 7. How It All Works Together (Container Example)

```
Process starts
    │
    ├─► Namespaces       → Isolation (own Net-NS, own PID-NS)
    ├─► cgroups          → Resource limiting (CPU, RAM, I/O)
    ├─► Seccomp          → Only allowed syscalls are permitted
    ├─► AppArmor/SELinux → Only allowed file accesses are permitted (MAC)
    ├─► Capabilities     → Only the capabilities the process needs
    └─► DAC (UID/GID)    → Unix permissions are checked
```

---

## 8. Baseline Hardening Guide (Enforcement)

How to actually turn the mechanisms above on. Commands assume Arch/Debian;
substitute the package manager where needed. Always test in `permissive`/`complain`
mode before enforcing in production.

### 8.1 Harden DAC / filesystem

```
    # Tighten sensitive files
    sudo chmod 600 /etc/shadow /etc/gshadow
    sudo chmod 644 /etc/passwd /etc/group

    # Sane default umask (027 = group r-x, other none)
    echo 'umask 027' | sudo tee /etc/profile.d/umask.sh

    # Review the setuid/setgid attack surface and strip what you don't need
    find / -perm -4000 -o -perm -2000 -type f 2>/dev/null
```

### 8.2 Enforce AppArmor (Debian/Ubuntu/Arch)

```
    sudo pacman -S apparmor            # or: apt install apparmor apparmor-utils
    # Ensure the LSM is active at boot (kernel cmdline):
    #   lsm=landlock,lockdown,yama,integrity,apparmor,bpf
    sudo systemctl enable --now apparmor

    aa-status                          # list profiles and their modes
    sudo aa-complain /path/to/profile  # log-only while tuning
    sudo aa-enforce  /path/to/profile  # enforce once tuned
```

### 8.3 Enforce SELinux (RHEL/Fedora)

```
    sestatus                           # current mode and policy
    sudo setenforce 1                  # runtime: switch to enforcing
    # Persist in /etc/selinux/config:  SELINUX=enforcing
    sudo ausearch -m avc -ts recent    # inspect denials before enforcing
    sudo audit2allow -a                # draft policy from denials (review first!)
```

### 8.4 Drop capabilities for services (systemd)

systemd units are the cleanest place to apply least-privilege. Add a drop-in:

```
    sudo systemctl edit myservice
```

```ini
[Service]
# Grant only what's needed
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=yes

# Sandboxing
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
RestrictAddressFamilies=AF_INET AF_INET6
SystemCallFilter=@system-service
SystemCallArchitectures=native
LockPersonality=yes
MemoryDenyWriteExecute=yes
```

```
    sudo systemctl daemon-reload && sudo systemctl restart myservice
    systemd-analyze security myservice   # score the unit's exposure
```

### 8.5 Tighten ptrace and kernel lockdown

```
    # yama: restrict ptrace to parent/child
    echo 'kernel.yama.ptrace_scope = 1' | sudo tee /etc/sysctl.d/10-ptrace.conf
    sudo sysctl --system

    # lockdown requires Secure Boot; confirm it is engaged
    cat /sys/kernel/security/lockdown     # e.g. [none] integrity confidentiality
```

### 8.6 Baseline checklist

- [ ] `/etc/shadow` `600`, `/etc/passwd` `644`; default `umask 027`
- [ ] setuid/setgid inventory reviewed, unneeded bits removed
- [ ] `sudo` rules are explicit, least-privilege, no stray `NOPASSWD`
- [ ] PAM enforces password quality and `pam_faillock` lockout
- [ ] AppArmor **enforcing** (or SELinux `enforcing`) with no unexplained denials
- [ ] Network services run under systemd sandboxing with a trimmed `CapabilityBoundingSet`
- [ ] `NoNewPrivileges=yes` + `SystemCallFilter` on exposed services
- [ ] `kernel.yama.ptrace_scope >= 1`
- [ ] `unprivileged_bpf_disabled = 1` (see [hardening.md](hardening.md))
- [ ] FIM (Wazuh syscheck / AIDE) watching critical paths for after-the-fact detection

> See [hardening.md](hardening.md) for the kernel-`sysctl`, SSH, firewall, and audit baseline that pairs with this file.
