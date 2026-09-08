# Windows Administration & Active Directory — Reference

## 1. Windows Architecture Basics

**Layers (user-mode → kernel-mode):**
- Win32 subsystem / API (`kernel32.dll`, `user32.dll`, `advapi32.dll`) — what most apps call
- `ntdll.dll` — thin wrapper, transitions to kernel via syscalls
- **Executive** (`ntoskrnl.exe`) — Object Manager, Process/Thread Manager (PsXxx), Memory Manager (MmXxx), I/O Manager, Security Reference Monitor (SRM), Config Manager (registry)
- **HAL** (Hardware Abstraction Layer) — sits between kernel and hardware
- Kernel-mode drivers (.sys) sit alongside the Executive; user-mode drivers exist via UMDF

**Key subsystems to know:**
- **Object Manager** — everything (files, processes, mutexes, registry keys) is a kernel object with a namespace (`\Device`, `\??`, `\Sessions`, etc.) — explore with WinObj (Sysinternals)
- **Security Reference Monitor (SRM)** — enforces access checks: compares a thread's **access token** (SIDs, privileges, integrity level) against an object's **Security Descriptor** (owner, DACL, SACL)
- **LSASS** (`lsass.exe`) — Local Security Authority Subsystem: handles logon, token creation, and caches credentials/Kerberos tickets (this is why it's the #1 credential-dumping target — mimikatz, etc.)
- **SAM** (Security Accounts Manager) — local user database, hive at `HKLM\SAM`, only accessible to SYSTEM
- **Session 0 isolation** — since Vista, services run in Session 0, user desktops start at Session 1+

**Processes worth knowing on a DC/server:**
| Process | Role |
|---|---|
| `lsass.exe` | Auth, token issuance, credential cache |
| `services.exe` (SCM) | Service Control Manager |
| `wininit.exe` | Spawns services.exe, lsass.exe |
| `svchost.exe` | Generic host for DLL-based services (grouped by `-k` group) |
| `csrss.exe` | Win32 client/server runtime |
| `smss.exe` | Session Manager (first user-mode process) |
| `winlogon.exe` | Logon UI, SAS (Ctrl+Alt+Del), desktop switching |
| `ntdsutil`/`ntds.dll` | AD database engine (on DCs) |

---

## 2. The Registry

**Purpose:** hierarchical database of config for the OS, drivers, services, and apps. Backing files ("hives") live in `C:\Windows\System32\config\` and per-user in `NTUSER.DAT`.

**Root keys:**
| Hive | Contents |
|---|---|
| `HKEY_LOCAL_MACHINE` (HKLM) | Machine-wide config. Subkeys: `SAM`, `SECURITY`, `SOFTWARE`, `SYSTEM`, `HARDWARE` (volatile) |
| `HKEY_CURRENT_USER` (HKCU) | Symlink into `HKEY_USERS\<SID>` for the logged-on user |
| `HKEY_USERS` (HKU) | All loaded user profiles by SID |
| `HKEY_CLASSES_ROOT` (HKCR) | File associations / COM registration — merge of `HKLM\SOFTWARE\Classes` and `HKCU\...\Classes` |
| `HKEY_CURRENT_CONFIG` | Active hardware profile (symlink into HKLM\SYSTEM) |

**HKLM\SYSTEM\CurrentControlSet — the important one for admins/forensics:**
- `Services\` — every driver and service, its start type (0=Boot,1=System,2=Auto,3=Manual,4=Disabled) and ImagePath
- `Control\Lsa` — security policy, auth packages, `RunAsPPL` (LSA protection)
- `Control\Terminal Server` — RDP settings
- Actual active set is `CurrentControlSet`, which is a symlink to `ControlSet001`/`002` (`Select\Current` tells you which)

**Common admin/forensic keys:**
- Autoruns: `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run(Once)`, same under HKCU
- Installed software: `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall`
- Network shares: `HKLM\SYSTEM\CurrentControlSet\Services\LanmanServer\Shares`
- RDP saved creds / recent docs / typed paths: various under `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer`
- Domain join info: `HKLM\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters`

**Data types:** `REG_SZ` (string), `REG_DWORD`/`REG_QWORD` (32/64-bit int), `REG_BINARY`, `REG_MULTI_SZ`, `REG_EXPAND_SZ` (contains `%ENV_VAR%`).

**Tools:** `regedit.exe` (GUI), `reg.exe` / `Get-ItemProperty`/`New-ItemProperty` (PowerShell, provider `HKLM:`/`HKCU:`), `reg save`/`reg load` for offline hive analysis (forensics — pull hives with a tool that bypasses the file lock, e.g. `reg save`, VSS, or KAPE).

---

## 3. Windows Server Roles (admin side)

Install/manage via Server Manager GUI or PowerShell (`Install-WindowsFeature` / `Add-WindowsFeature`).

| Role | Purpose |
|---|---|
| AD DS | Active Directory Domain Services — the directory itself |
| AD CS | Certificate Services (PKI) — also a major modern AD attack surface (ESC1-ESC16 etc.) |
| AD FS | Federation Services — SSO/claims-based auth, SAML/WS-Fed |
| DNS Server | AD is DNS-dependent; AD-integrated zones replicate via AD |
| DHCP Server | Often co-located with a DC in small environments |
| File and Storage Services | SMB shares, DFS |
| Remote Desktop Services | RDS/terminal services farm |
| Network Policy Server (NPS) | RADIUS |
| WSUS | Patch management |

---

## 4. Active Directory Fundamentals

### Logical structure
- **Forest** — top-level security boundary, one or more domains, shares schema + global catalog
- **Domain** — administrative/replication boundary, has its own DC(s) and its own domain-level GPOs
- **OU (Organizational Unit)** — container for organizing objects and scoping GPOs/delegation (NOT a security boundary — permissions are)
- **Trusts** — link forests/domains (one-way/two-way, transitive/non-transitive); trust direction is opposite of access direction (trusting domain grants access to trusted domain's principals)

### Objects
- Users, Computers, Groups, GPOs, Contacts, OUs — all stored as objects with a `distinguishedName` (DN), `objectGUID`, `objectSID`, in the NTDS.dit database (`%SystemRoot%\NTDS\ntds.dit` on a DC)
- **Group types:** Security (used in ACLs/tokens) vs Distribution (email only)
- **Group scope:** Domain Local (assign resource permissions, can contain members from any trusted domain) → Global (contains accounts from same domain, used across domains) → Universal (forest-wide, members from any domain, stored in Global Catalog)

### FSMO roles (5 roles, single points of coordination — no true multi-master for these)
| Role | Scope | Function |
|---|---|---|
| Schema Master | Forest | Only DC allowed to modify the schema |
| Domain Naming Master | Forest | Adding/removing domains from forest |
| RID Master | Domain | Allocates RID pools to DCs so SIDs never collide |
| PDC Emulator | Domain | Time sync source, password change authority, GPO conflict tiebreaker, legacy NT4 BDC emulation |
| Infrastructure Master | Domain | Fixes cross-domain object references (group memberships) — must NOT be on a Global Catalog server unless all DCs are GCs |

`netdom query fsmo` or `Get-ADForest` / `Get-ADDomain` (PowerShell) to check.

### Replication
- Multi-master (except FSMO roles) via **DRS/RPC**, using **USN (Update Sequence Numbers)** + a **high-watermark table** and **up-to-dateness vector** to avoid replaying changes
- **KCC (Knowledge Consistency Checker)** auto-builds the replication topology; **ISTG** does it between sites
- **Sites & Subnets** define replication scheduling — intra-site replication is fast/frequent, inter-site is scheduled (SMTP or IP transport) to save WAN bandwidth
- **SYSVOL** replication (GPOs, scripts) uses DFSR (or legacy FRS)
- Object deletion → **tombstone** (kept `tombstoneLifetime`, default 180 days) → then garbage collected

### Global Catalog (GC)
- Partial, read-only, attribute-subset replica of *every* domain in the forest, held on GC-designated DCs
- Needed for forest-wide searches, UPN logon, Universal group membership resolution

### Group Policy
- GPOs = SYSVOL (files: `GptTmpl.inf`, ADM(X) templates) + AD GPC (versioning metadata) two halves that must stay in sync
- **Processing order (last writer wins, mnemonic LSDOU):** Local → Site → Domain → OU (closest OU to object applied last = wins on conflict)
- Inheritance can be blocked (Block Inheritance on OU) or forced (Enforced on the GPO link) — Enforced beats Block
- **Client-side extensions (CSEs)** actually apply settings; `gpupdate /force` to reapply, `gpresult /r` or `/h report.html` to see Resultant Set of Policy (RSoP)
- Security-relevant: Group Policy Preferences (GPP) `cpassword` in SYSVOL was a classic cred-leak vuln (MS14-025) — still relevant on old/lab environments

### Authentication protocols
**Kerberos (default since Win2000):**
1. **AS-REQ** → client sends pre-auth (timestamp encrypted with user's NTLM hash) to KDC (on DC)
2. **AS-REP** → KDC returns **TGT** (encrypted with `krbtgt` account's hash) + session key
3. **TGS-REQ** → client presents TGT to KDC, asks for a service ticket for a specific SPN
4. **TGS-REP** → KDC returns **service ticket** (encrypted with the *service account's* hash)
5. **AP-REQ** → client presents service ticket directly to the target service, which decrypts it with its own hash — no DC contact needed at this step

Why this matters offensively/defensively: golden tickets forge step 2 (need krbtgt hash), silver tickets forge step 4 (need service account hash), kerberoasting abuses step 3/4 (request a ticket for any SPN, crack it offline since it's encrypted with the service account's password-derived key), AS-REP roasting abuses accounts with `DONT_REQUIRE_PREAUTH` set (skip step 1's proof).

**NTLM (legacy, still used as fallback / for non-domain / local auth):** challenge-response, no third party involved for local, DC acts as validator for domain (Netlogon "pass-through" auth). Weaker — no mutual auth, vulnerable to relay attacks (NTLM relay).

**LDAP** — the protocol used to actually query/modify AD objects. Port 389 (636 for LDAPS/TLS, 3268/3269 for GC). LDAP signing/channel binding matter for relay-resistance.

---

## 5. PowerShell Admin Cheat Sheet

```powershell
# AD module (RSAT: Import-Module ActiveDirectory)
Get-ADUser -Filter * -Properties *
Get-ADUser -Identity jdoe -Properties MemberOf
New-ADUser -Name "Jane Doe" -SamAccountName jdoe -Enabled $true -AccountPassword (Read-Host -AsSecureString)
Set-ADAccountPassword -Identity jdoe -Reset -NewPassword (ConvertTo-SecureString "P@ss" -AsPlainText -Force)
Get-ADGroupMember -Identity "Domain Admins" -Recursive
Add-ADGroupMember -Identity "IT Staff" -Members jdoe
Get-ADComputer -Filter * -Properties OperatingSystem
Get-ADDomainController -Filter *
Get-ADReplicationFailure -Target (Get-ADDomainController).HostName
Search-ADAccount -AccountInactive -TimeSpan 90.00:00:00 -UsersOnly

# GPO
Get-GPO -All
New-GPLink -Name "Disable USB" -Target "OU=Workstations,DC=corp,DC=local"
gpresult /h C:\rsop.html /f

# Local accounts / groups (non-domain)
Get-LocalUser; Get-LocalGroupMember Administrators
net user /domain; net group "Domain Admins" /domain

# Events / diagnostics
Get-WinEvent -LogName Security -MaxEvents 50
Get-EventLog -LogName System -Newest 20
```

---

## 6. Hardening Checklist (DC-focused)

- Tiered administration model (Tier 0 = DCs/AD CS/AD FS, Tier 1 = servers, Tier 2 = workstations) — Tier 0 admins never log onto lower tiers
- **LAPS** (Local Administrator Password Solution) for unique, rotated local admin passwords
- Disable NTLMv1, enforce SMB signing and LDAP signing/channel binding
- Protect `krbtgt` — rotate its password twice (with replication delay between) after any suspected compromise
- Restrict Kerberoasting exposure: use `gMSA`/`dMSA` (managed service accounts, random 240-char passwords) instead of user accounts for SPNs
- Enable `Credential Guard` (isolates LSASS secrets in a VBS enclave) and LSA protection (`RunAsPPL`)
- Audit `AdminSDHolder`/`SDProp` protected group membership regularly
- Least privilege on AD CS templates (watch for ESC1/ESC8-style misconfigurations if AD CS is deployed)
- Enable and forward Advanced Audit Policy (esp. 4624/4625/4768/4769/4776/4670/5136) to a SIEM

---

## 7. Practice Roadmap

1. **Concepts** — HTB Academy: *Windows Fundamentals* → *Introduction to Active Directory* → *Active Directory LDAP* → *Active Directory Enumeration & Attacks* (fundamental-tier modules are accessible on the free plan; deeper AD modules require a paid Academy tier)
2. **Guided practice** — HTB *Starting Point* (all three tiers are free on every account) — Tier 2 has your first real Windows/SMB/AD-adjacent boxes
3. **Unguided practice** — HTB *Active Machines* (the 20 currently-live boxes, free for everyone) — filter by OS = Windows; an AD box usually means SMB/LDAP/Kerberos ports (135, 139, 445, 389, 464, 3268, 88) in the nmap scan
4. **Deeper AD chains** — once you outgrow the free rotation, VIP+ unlocks the retired catalog, which includes the classic beginner AD boxes (Active, Forest, Sauna, Monteverde, Blackfield, etc.) that most write-ups reference
5. **Full network practice** — HTB Pro Labs (Offshore/Dante for multi-machine AD networks) once single-box AD attack chains feel comfortable

---

## 8. Core References
- Microsoft Learn: "Active Directory Domain Services Overview"
- Microsoft Learn: "Kerberos Authentication Overview"
- Sysinternals Suite (Process Explorer, Autoruns, Sysmon) for live inspection
- `adexplorer`/`BloodHound` for visualizing AD relationships and attack paths once you get into the AD Enumeration module
