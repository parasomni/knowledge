# Access Controls: Windows

> Companion note: [Access Controls: Linux](../linux-hardening/access_controls_linux.md)

## Overview

| Layer | Mechanism | Purpose |
|---|---|---|
| **DAC** | Access Token + DACL/SACL (per-object ACLs) | Identity-based "who may do what" |
| **Privileges** | Token privileges (`SeDebugPrivilege`, …) | Powers independent of object ACLs |
| **MAC** | Mandatory Integrity Control (Integrity Levels) + UIPI | No write-up across trust boundaries |
| **Elevation** | UAC (User Account Control) | Split admin/standard token, consent prompt |
| **Isolation** | AppContainer, Job Objects, Sessions, Sandbox | Confine and separate processes |
| **App control** | AppLocker, WDAC (App Control for Business) | Allow-list which code may run |
| **Credential** | Credential Guard, LSA protection, LAPS | Protect secrets and local admin passwords |

All layers complement each other — no single mechanism is sufficient on its own.

---

## 1. DAC – Discretionary Access Control

- Every process carries an **Access Token** containing: User-SID, Group-SIDs, Privileges, Integrity Level
- On every object access (file, registry key, named pipe, service, …) the token is
  checked against the object's **DACL** (Discretionary ACL)
- → "Who is allowed to do what?" based on identity (SIDs)

| Term | Meaning |
|---|---|
| **SID** | Security Identifier — unique ID for a user/group/computer |
| **DACL** | Discretionary ACL — the allow/deny ACEs that grant or refuse access |
| **SACL** | System ACL — defines what accesses get *audited* (not access control itself) |
| **ACE** | Access Control Entry — one rule inside an ACL |

```
    whoami /all                        # SIDs, groups, privileges, integrity level
    icacls C:\path\to\file             # view a file's DACL
    icacls C:\path /grant User:(R)     # grant read
    icacls C:\path /remove User        # remove an ACE
```

---

## 2. Privileges – Token Rights

Privileges are powers held *in the token*, checked independently of object ACLs.
They are the Windows analogue of Linux capabilities, and several are effectively
admin-equivalent — a standing privilege audit matters.

| Privilege | Grants (and why it's dangerous) |
|---|---|
| `SeDebugPrivilege` | Open any process — read/inject LSASS, dump credentials |
| `SeBackup`/`SeRestore` | Read/write any file bypassing the DACL |
| `SeTakeOwnership` | Seize ownership of any object |
| `SeImpersonate`/`SeAssignPrimaryToken` | Impersonate other tokens (potato-style escalation) |
| `SeLoadDriver` | Load a kernel driver (BYOVD attacks) |

```
    whoami /priv                       # privileges held by the current token
```

> Enabled service accounts holding `SeImpersonatePrivilege` are a classic local
> privilege-escalation path — review who holds it.

---

## 3. MAC – Mandatory Integrity Control (MIC)

- Implemented via **Integrity Levels** (Untrusted, Low, Medium, High, System)
- Enforces **no write-up**: a Low-IL process (a sandboxed browser tab) cannot write
  into a Medium/High-IL process or object
- **UIPI** (User Interface Privilege Isolation) blocks lower-IL processes from
  sending window messages to higher-IL ones (shatter-attack defense)
- It is an *additional* mechanism on top of the Access Token, **not** a replacement
- True Bell-LaPadula MAC only exists in special variants (Windows MLS mode)

```
    icacls C:\path\file /setintegritylevel Low   # label an object Low-IL
    whoami /groups                               # shows the token's mandatory label
```

> **Important:** Access Tokens alone are DAC, not MAC. MAC is introduced through
> Integrity Levels (and enforced by MIC/UIPI).

---

## 4. UAC – User Account Control

- An administrator logs on with a **split token**: a filtered standard-user token
  for normal work and a full-admin token released only on elevation
- Elevation prompts on the **Secure Desktop**; High-IL is granted to the elevated process
- UAC is a convenience boundary, **not** a hard security boundary per Microsoft —
  but set it to "Always notify" and never disable it

---

## 5. Isolation & Sandboxing

| Mechanism | What it isolates |
|---|---|
| **AppContainer** | Low-IL sandbox with explicit capability grants (UWP apps, Edge, Defender Application Guard) |
| **Job Objects** | Group of processes with shared limits (CPU, memory, UI restrictions) |
| **Sessions** | Session 0 isolation separates services from interactive user desktops |
| **Windows Sandbox** | Disposable, hardware-virtualized throwaway Windows instance |
| **Virtualization-Based Security (VBS)** | Hyper-V-backed trust boundary underpinning Credential Guard & HVCI |

---

## 6. Application Control

- **AppLocker** — allow-list rules (publisher / path / hash) for exe, scripts, MSI, DLLs
- **WDAC / App Control for Business** — kernel-enforced code-integrity allow-listing,
  stronger than AppLocker and harder to bypass
- **Attack Surface Reduction (ASR)** rules — block common malware behaviors
  (Office spawning children, credential theft from LSASS, etc.)

---

## 7. Credential & Kernel Protection

| Feature | Protects |
|---|---|
| **Credential Guard** | Isolates LSASS secrets (NTLM hashes, Kerberos TGTs) in a VBS enclave — defeats most `mimikatz` dumping |
| **LSA Protection (RunAsPPL)** | Marks LSASS as a protected process so non-PPL code can't read it |
| **LAPS** | Randomizes and rotates the local Administrator password per machine |
| **HVCI** | Hypervisor-enforced code integrity — only signed drivers/kernel code run |

---

## 8. Baseline Hardening Guide (Enforcement)

How to actually turn the mechanisms above on. Prefer **Group Policy** (`gpedit.msc`)
or Intune for fleets; the commands below are for standalone hosts and quick checks.
Test in audit/log-only mode before enforcing.

### 8.1 Start from a vendor baseline

- Microsoft **Security Compliance Toolkit** ships GPO baselines for Windows and Office —
  import these rather than hand-rolling every setting
- CIS Benchmarks provide an equivalent audited baseline

### 8.2 Tighten UAC & audit standing privileges

```
    # UAC: Always notify (via Group Policy, "User Account Control:" settings)
    # Confirm no stray accounts hold dangerous privileges:
    whoami /priv
    # Remove via: Group Policy > Security Settings > Local Policies > User Rights Assignment
```

### 8.3 Enable auditing (SACL / audit policy)

```
    auditpol /get /category:*                         # current audit config
    auditpol /set /subcategory:"Logon" /success:enable /failure:enable
    auditpol /set /subcategory:"Process Creation" /success:enable
    # Also enable command-line auditing (GPO) + PowerShell Script Block Logging
```

### 8.4 Turn on credential protection

```
    # Credential Guard & HVCI require VBS (UEFI, Secure Boot, TPM 2.0).
    # Enable via Group Policy: "Turn On Virtualization Based Security"
    #   - Credential Guard: Enabled with UEFI lock
    #   - HVCI (Memory Integrity): Enabled

    # LSA Protection (RunAsPPL):
    reg add HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v RunAsPPL /t REG_DWORD /d 1 /f

    # Deploy Windows LAPS for local-admin password rotation (built in on modern Windows).
```

### 8.5 Enforce application control

```
    # AppLocker: start in Audit Only, generate default rules, review the event log,
    # then switch each rule collection to "Enforce rules".
    #   gpedit.msc > Application Control Policies > AppLocker

    # WDAC: build a policy from a known-good reference image, deploy in audit mode,
    # review CodeIntegrity events, then enforce.

    # Attack Surface Reduction rules (audit first, then block):
    Set-MpPreference -AttackSurfaceReductionRules_Ids <ruleGUID> -AttackSurfaceReductionRules_Actions AuditMode
```

### 8.6 Label low-trust content / least privilege

```
    # Run risky content Low-IL where applicable
    icacls "C:\Downloads\*" /setintegritylevel (OI)(CI)Low

    # Principle: users are standard users, not local admins; admin work uses a
    # separate privileged account (tiered admin model).
```

### 8.7 Baseline checklist

- [ ] Microsoft/CIS security baseline imported as the starting GPO set
- [ ] Users run as **standard** users; admin tasks use separate, tiered accounts
- [ ] UAC set to **Always notify**, never disabled
- [ ] `whoami /priv` reviewed — no service/user holds `SeDebug`/`SeImpersonate`/`SeBackup` unnecessarily
- [ ] Credential Guard + **LSA Protection (RunAsPPL)** + HVCI enabled (VBS available)
- [ ] **LAPS** rotating local Administrator passwords
- [ ] AppLocker **or** WDAC enforcing an allow-list (after an audit phase)
- [ ] ASR rules in Block mode (after audit)
- [ ] Audit policy on: logon, process creation (with command line), object access, privilege use
- [ ] BitLocker + Secure Boot + TPM 2.0 for boot/disk integrity
- [ ] EDR/SIEM collecting the audit + Script Block logs for after-the-fact detection

> The Windows integrity model (DAC + MIC + UAC) is the counterpart to the Linux
> DAC + LSM + capabilities stack described in
> [access_controls_linux.md](../linux-hardening/access_controls_linux.md).
