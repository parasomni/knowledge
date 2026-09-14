# OpenBSD — Architecture & Design Philosophy

OpenBSD is a BSD-derived, Unix-like operating system whose entire design is organized around one goal: **correctness and security first, features second**. Nearly every architectural decision — from the kernel to the base system to the release process — traces back to that priority.

---

## 1. Project Philosophy

- **"Secure by default."** A freshly installed system should be safe to expose to a network with minimal hardening required.
- **Correctness over features.** Code is expected to be simple, auditable, and provably correct rather than maximally capable.
- **Full disclosure & proactive security.** Bugs are fixed even without a known exploit; entire classes of bugs are eliminated proactively rather than patched reactively.
- **One base system, one team.** Unlike Linux, the kernel, core utilities, compiler toolchain integration, and most daemons (httpd, ntpd, smtpd, sshd, pf) are developed and released together as a single coherent system — not assembled from independently versioned upstream projects.
- **Code auditing.** Historically, the OpenBSD team performed systematic line-by-line audits of the source tree, which produced techniques (privilege separation, `strlcpy`/`strlcat`) later adopted across the industry.

---

## 2. Kernel Design

- **Monolithic kernel** with loadable device drivers, similar lineage to 4.4BSD, NetBSD, and FreeBSD.
- **Single-CPU-focused historically**, with multiprocessor (MP) support added conservatively and audited carefully rather than rushed — OpenBSD prioritizes a correct locking model over maximum SMP throughput.
- **W^X (Write XOR Execute).** Memory pages are never simultaneously writable and executable, enforced system-wide, including for the kernel and by default for userland processes.
- **KARL (Kernel Address Randomized Link).** Every boot re-links the kernel with randomized layout, so kernel addresses differ across reboots on the same machine — defeats a whole class of remote kernel exploitation that assumes a fixed layout.
- **Pledge and unveil** (see below) are enforced at the syscall layer, not bolted on as a separate security module — this keeps the trusted computing base small compared to Linux's LSM/SELinux/AppArmor stack.
- **No loadable kernel module security bypass surface by default** in the way Linux's LKM subsystem can be abused — driver support is more limited, but the attack surface is smaller.

---

## 3. Security Mitigations Built Into the Base System

| Mitigation | Purpose |
|---|---|
| `pledge(2)` | Restricts a process to a declared subset of syscalls for the rest of its life. A compromised process (e.g., via a parsing bug) cannot escalate to syscalls it never needed. |
| `unveil(2)` | Restricts the filesystem paths a process may see or access at all — paths not unveiled do not exist from the process's point of view. |
| `W^X` | No page is writable and executable at once; blocks classic shellcode injection. |
| `ASLR` | Randomizes stack, heap, mmap, and shared library base addresses. |
| Stack Protector / `RETGUARD` | Compiler-inserted return-address integrity checks (RETGUARD) on top of stack canaries, specifically targeting ROP-style attacks. |
| `arc4random(3)` | A high-quality, always-available CSPRNG built into libc — used pervasively instead of ad hoc or weak PRNGs. |
| Privilege separation | Long-running daemons (`sshd`, `httpd`, `smtpd`, `bgpd`) split into a small privileged monitor process and unprivileged workers, so a worker compromise doesn't grant root. |
| Privilege revocation / chroot | Daemons drop privileges and `chroot(2)` into a jail directory as soon as privileged setup is complete. |

The philosophy: **assume bugs exist**, and structure the system so an individual bug's blast radius is contained rather than trying to eliminate all bugs through review alone.

---

## 4. Base System vs. Ports

- **Base system**: kernel, libc, core utilities, compiler toolchain hooks, and first-party daemons (`httpd`, `relayd`, `smtpd`, `ntpd`, `sshd`, `pf`) are maintained *in the same source tree*, released together, versioned together. This avoids the dependency-hell and inconsistent-hardening problems of distributions that bundle many independently maintained upstream components.
- **Ports/packages**: third-party software lives in the separate ports tree, built into binary packages (`pkg_add`). This keeps the security-critical base minimal while still providing a large software ecosystem.
- A strict base/ports separation means the security guarantees above apply to daemons in base; ported software carries whatever hardening upstream provides plus OpenBSD's system-wide mitigations (W^X, ASLR, etc.), but not necessarily pledge/unveil unless the port has been patched.

---

## 5. Release Model

- **Fixed 6-month release cycle** (e.g., 7.4, 7.5), each supported for two release cycles (~1 year) via binary patches (`syspatch`) and source patches.
- **-current** is the constantly-updated development branch; **-release** is the frozen, tested snapshot.
- Every release ships with a detailed, human-written **CHANGELOG** and upgrade guide — upgrades are deliberately a manual, well-documented process rather than a fully automated rolling update, reinforcing the "understand your system" philosophy.

---

## 6. Networking Stack & pf

- **pf (Packet Filter)** is OpenBSD's native firewall/NAT engine, tightly integrated into the kernel network stack, and is the reference implementation later ported to FreeBSD and other BSDs.
- Network stack design favors clarity and correctness in the code path over maximum throughput tuning knobs — routing, NAT, queuing (`ALTQ`/newer bandwidth control), and packet filtering are one coherent subsystem rather than separate frameworks (contrast with Linux's iptables/nftables/tc split).

---

## 7. Filesystem & Storage

- Default filesystem is **FFS (Fast File System)**, a BSD UFS derivative, with soft updates for crash consistency (journaling is not used the way Linux ext4/xfs use it).
- Full-disk encryption is provided via **softraid crypto** discipline, integrated into the `bioctl` storage framework rather than a separate LVM+LUKS-style stack.

---

## 8. Design Trade-offs to Understand

- **Hardware support is intentionally narrower** than Linux — drivers must meet the same code-quality and auditability bar as the rest of the base system, so bleeding-edge or vendor-blob-dependent hardware is often unsupported.
- **Performance is not the primary optimization target** — mitigations like W^X, RETGUARD, and KARL cost some performance in exchange for eliminated bug classes.
- **Small team, small trusted base** — this enables the auditing-driven model but means feature velocity is deliberately slower than Linux.

Understanding OpenBSD means recognizing that nearly every subsystem choice (kernel, daemons, network stack, filesystem) is downstream of the same underlying value: keep the trusted computing base small, auditable, and resistant to entire bug classes, even at the cost of raw feature count or performance.
