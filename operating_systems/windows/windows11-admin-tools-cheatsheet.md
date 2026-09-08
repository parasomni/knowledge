# Windows 11 — Administrative Tools Cheatsheet

How to open every important admin/config tool: via `Win+R` (Run command), via Settings app path, or via `mmc.exe` snap-ins. Most classic tools still open a normal window even though the new Settings app is the "front door" for most consumer-facing options.

**Fastest way to open almost anything:** `Win+X` (Quick Link / power-user menu) lists most of the tools below directly. `Win+R` runs any command in the table. Typing the command into the Start menu search box also works for all of them.

---

## 1. Core Management Consoles (MMC snap-ins)

All of these are `.msc` files — run via `Win+R`. Most need admin rights for changes (right-click → Run as administrator, or they'll UAC-prompt automatically).

| Tool | Command | What it manages |
|---|---|---|
| Computer Management | `compmgmt.msc` | One console bundling: Task Scheduler, Event Viewer, Shared Folders, Local Users & Groups, Performance, Device Manager, Disk Management, Services & Applications |
| Event Viewer | `eventvwr.msc` | Windows/Application/Security/Setup/System logs, custom views, task scheduling on events |
| Device Manager | `devmgmt.msc` | Hardware devices, drivers, driver rollback/update, resource conflicts |
| Disk Management | `diskmgmt.msc` | Partitions, volumes, formatting, disk initialization, mount points |
| Services | `services.msc` | Start/stop/disable services, set startup type, recovery actions, service account |
| Local Users and Groups | `lusrmgr.msc` | Local accounts and groups (**Windows 11 Pro/Enterprise only** — not on Home) |
| Local Group Policy Editor | `gpedit.msc` | Local machine policy (**Pro/Enterprise/Education only**) |
| Local Security Policy | `secpol.msc` | Password policy, account lockout, audit policy, user rights assignment, local security options (**Pro/Enterprise only**) |
| Certificate Manager (current user) | `certmgr.msc` | Personal, Trusted Root CA, Intermediate CA certificate stores for the logged-in user |
| Certificates (computer account) | via `mmc.exe` → Add Snap-in → Certificates → Computer account | Machine-wide cert stores (needed for server certs, code signing trust, etc.) |
| Certification Authority (if role installed) | `certsrv.msc` | Manage an actual issuing CA — only relevant if the box runs AD CS |
| Task Scheduler | `taskschd.msc` | Scheduled tasks, triggers, actions, history |
| Print Management | `printmanagement.msc` | Printers, drivers, print queues across multiple servers (**Pro/Enterprise**) |
| Performance Monitor | `perfmon.msc` | Real-time/logged performance counters, Data Collector Sets |
| Resource Monitor | `resmon.exe` (not .msc) | Live CPU/memory/disk/network breakdown per process |
| Windows Firewall with Advanced Security | `wf.msc` | Inbound/outbound rules, connection security rules, profiles (Domain/Private/Public) |
| Disk Cleanup | `cleanmgr.exe` | Temp file / system cleanup |
| Component Services | `comexp.msc` | COM+/DCOM applications, transaction management |
| WMI Control | `wmimgmt.msc` | WMI namespace security and configuration |
| Volume Activation Tools | `slui.exe` | Windows activation status/license key entry |
| System Configuration | `msconfig.exe` | Boot options, startup items (legacy — mostly superseded by Task Manager's Startup tab and Settings) |

**Build your own console:** `mmc.exe` → File → Add/Remove Snap-in → pick multiple → save as a custom `.msc` for a one-stop admin console.

---

## 2. Control Panel Applets (still present in Win11, hidden behind Settings)

Control Panel isn't gone — many applets have no full Settings equivalent yet. Open Control Panel itself with `control`, or jump straight to an applet:

| Applet | Command | Notes |
|---|---|---|
| Control Panel (home) | `control` | |
| Programs and Features | `appwiz.cpl` | Uninstall/change desktop apps (not Store apps) |
| Network Connections | `ncpa.cpl` | Per-adapter properties, IPv4/IPv6 config, adapter enable/disable |
| System Properties | `sysdm.cpl` | Computer name, domain/workgroup join, hardware profiles, System Protection (Restore Points), Remote settings, Advanced (env vars, performance, startup/recovery) |
| Power Options | `powercfg.cpl` | Power plans, sleep/hibernate thresholds, button behavior |
| Sound | `mmsys.cpl` | Playback/recording devices, levels, default device |
| Date and Time | `timedate.cpl` | Clock, time zone, internet time sync (NTP server) |
| Region | `intl.cpl` | Locale, formats, keyboard layouts |
| Mouse | `main.cpl` | Pointer speed, buttons, wheel |
| Fonts | `fonts` (or `control fonts`) | Installed fonts |
| Windows Defender Firewall (basic) | `firewall.cpl` | Simplified firewall UI (advanced = `wf.msc`) |
| User Accounts | `netplwiz` (user autologon/advanced) or `control userpasswords2` | Manage local accounts, auto-login, "Users must enter a password" toggle |
| Internet Options | `inetcpl.cpl` | Legacy IE engine settings, still backs some system-wide proxy/cert settings used by other apps |
| Device Manager (applet form) | `hdwwiz.cpl` | Same as devmgmt.msc |
| Add Hardware Wizard | `hdwwiz.exe` | Legacy hardware detection |
| Ease of Access Center | `control access.cpl` | Legacy accessibility settings (mostly moved to Settings) |
| Storage Spaces | `control /name Microsoft.StorageSpaces` | Software RAID-like pooling |
| BitLocker | `control /name Microsoft.BitLockerDriveEncryption` | Same as Settings > Privacy & Security > Device Encryption but with full options (Pro/Enterprise for full BitLocker; Home only gets Device Encryption) |
| Credential Manager | `control /name Microsoft.CredentialManager` | Saved Windows/Web credentials |
| Recovery | `control /name Microsoft.Recovery` | Reset this PC, Advanced startup |

---

## 3. Settings App (`ms-settings:` URI scheme) — deep links

The Settings app (`Win+I`) is organized in pages; you can jump straight to any page with `Win+R` → `ms-settings:<page>`.

| Area | Command | Covers |
|---|---|---|
| System overview | `ms-settings:` | Landing page |
| Display | `ms-settings:display` | Resolution, scaling, multiple monitors, HDR |
| Sound | `ms-settings:sound` | New unified sound settings |
| Notifications | `ms-settings:notifications` | Per-app notification rules |
| Power & battery | `ms-settings:powersleep` | Sleep timers, battery saver |
| Storage | `ms-settings:storagesense` | Storage Sense, disk usage by category |
| Network & internet | `ms-settings:network-status` | Wi-Fi, Ethernet, VPN, Proxy, DNS |
| Bluetooth & devices | `ms-settings:bluetooth` | Pairing, device list |
| Windows Update | `ms-settings:windowsupdate` | Update history, pause updates, optional updates |
| Apps & features | `ms-settings:appsfeatures` | Modern install/uninstall list (Store + desktop) |
| Optional features | `ms-settings:optionalfeatures` | Language packs, RSAT tools, legacy components |
| Default apps | `ms-settings:defaultapps` | File type/protocol associations |
| Accounts | `ms-settings:yourinfo` | Account type, sign-in options, family accounts |
| Sign-in options | `ms-settings:signinoptions` | Windows Hello, PIN, security key |
| Windows Security (Defender) | `ms-settings:windowsdefender` or `windowsdefender:` | Virus/threat protection, firewall UI front-end, device security |
| Privacy & security | `ms-settings:privacy` | App permissions, telemetry, activity history |
| For Developers | `ms-settings:developers` | Developer Mode, sideloading, PowerShell/CMD default terminal |
| About | `ms-settings:about` | Build number, device specs, Rename this PC, Domain/workgroup join |
| Remote Desktop | `ms-settings:remotedesktop` | Enable/disable inbound RDP |

---

## 4. Terminal / Scripting-based Admin

| Tool | Launch | Use for |
|---|---|---|
| Windows Terminal | `wt` | Modern host for PowerShell/CMD/WSL tabs |
| PowerShell (7+/5.1) | `pwsh` / `powershell` | Primary modern admin shell — must "Run as administrator" for privileged cmdlets |
| Command Prompt | `cmd` | Legacy shell, still needed for some net/wmic/reg one-liners |
| Windows Subsystem for Linux | `wsl` | Linux userland alongside Windows |
| Registry Editor | `regedit` | GUI registry browser/editor |
| Group Policy Editor (domain-joined) | `gpedit.msc` (local) / RSAT GPMC (`gpmc.msc`, domain) | Domain GPO management needs RSAT: Group Policy Management installed |
| System File Checker | `sfc /scannow` (elevated cmd) | Repairs protected system files |
| DISM | `DISM /Online /Cleanup-Image /RestoreHealth` (elevated) | Repairs the Windows image/component store |
| Task Manager | `taskmgr` | Processes, Performance, Startup apps, Users, Details, Services tabs |
| Resource Monitor | `resmon` | Deep per-process resource view |
| Windows Memory Diagnostic | `mdsched` | RAM test on next reboot |
| System Information | `msinfo32` | Full hardware/driver/software inventory snapshot |
| Direct X Diagnostic Tool | `dxdiag` | GPU/DirectX/sound diagnostics |

---

## 5. Networking-specific Tools

| Tool | Command | Purpose |
|---|---|---|
| Network Connections | `ncpa.cpl` | Adapter settings (IP, DNS, adapter properties) |
| Network and Sharing Center (legacy) | `control /name Microsoft.NetworkAndSharingCenter` | Legacy view, still reachable |
| Windows Firewall (advanced) | `wf.msc` | Rule-level firewall config |
| ipconfig | `ipconfig /all` (cmd/PowerShell) | IP config, DNS, DHCP lease info |
| netstat | `netstat -ano` | Active connections/listening ports mapped to PID |
| Network diagnostics | `ms-settings:network-status` → Network troubleshooter | Automated repair wizard |
| Proxy settings | `ms-settings:network-proxy` | System-wide proxy config |
| Remote Desktop connection | `mstsc` | RDP client |
| Hyper-V Manager (if enabled) | `virtmgmt.msc` | VM management (**Pro/Enterprise, requires enabling the optional feature**) |

---

## 6. Certificate & PKI Tools

| Tool | Command | Purpose |
|---|---|---|
| Certificates (current user) | `certmgr.msc` | View/import/export personal & trust store certs for your profile |
| Certificates (local machine) | `mmc.exe` → Add Snap-in → Certificates → Computer account | Machine-wide cert stores — needed for services, RDP certs, etc. |
| certutil | `certutil -store My` (cmd) | Command-line cert store inspection/management, also does hashing (`certutil -hashfile`) |
| Certification Authority console | `certsrv.msc` | Only if the machine hosts an actual CA role |

---

## 7. Device & Driver Management

| Tool | Command | Purpose |
|---|---|---|
| Device Manager | `devmgmt.msc` | Enable/disable/update/roll back/uninstall drivers, view resource usage |
| Bluetooth & devices (Settings) | `ms-settings:bluetooth` | Pairing new devices |
| Printers & scanners | `ms-settings:printers` | Add/remove/manage printers |
| Device installation settings | `ms-settings:deviceinstall` | Auto driver download policy |
| USB/Removable storage policy | via Local Group Policy or `regedit` (`HKLM\SYSTEM\CurrentControlSet\Services\UsbStor`) | Restrict/enable USB storage |

---

## 8. Admin-only Elevated Utilities Worth Knowing

| Tool | Command | Purpose |
|---|---|---|
| Local Users and Groups | `lusrmgr.msc` | Create/disable/reset local accounts, group membership |
| Computer Management → Shared Folders | `fsmgmt.msc` | View/create SMB shares, open sessions, open files |
| Group Policy Result | `rsop.msc` | Read-only view of Resultant Set of Policy for the current user/machine |
| Group Policy Update | `gpupdate /force` (cmd/PowerShell) | Force-apply latest GPOs |
| Group Policy Report | `gpresult /h report.html` | Export applied policy as HTML |
| Windows Memory Diagnostic | `mdsched.exe` | Schedule a RAM test |
| Backup and Restore (legacy) | `sdclt.exe` | Legacy Windows 7-style backup, still present |
| File History | `ms-settings:backup` | Modern file versioning backup |
| System Restore | `rstrui.exe` | Roll back system state to a restore point |
| Optional Features / RSAT | `ms-settings:optionalfeatures` or `Add-WindowsCapability -Online -Name Rsat...` | Install admin tools for managing remote AD/DNS/DHCP/GPO from a Win11 client |

---

## 9. RSAT — Remote Server Administration Tools (for managing a domain from Win11)

Not installed by default; add via Settings → Optional Features → "Add a feature" → search RSAT, or PowerShell:
```powershell
Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online
```
This unlocks (among others): `dsa.msc` (Active Directory Users and Computers), `dssite.msc` (AD Sites and Services), `domain.msc` (AD Domains and Trusts), `dnsmgmt.msc` (DNS Manager), `dhcpmgmt.msc` (DHCP Manager), `gpmc.msc` (Group Policy Management Console) — the actual domain-admin toolkit, run from a joined client instead of RDPing into a DC.

| Tool | Command | Purpose |
|---|---|---|
| Active Directory Users and Computers | `dsa.msc` | User/group/OU management, delegation |
| Active Directory Sites and Services | `dssite.msc` | Site/subnet/replication link config |
| Active Directory Domains and Trusts | `domain.msc` | Trust relationships, forest functional level |
| Active Directory Administrative Center | `dsac.exe` | Modern GUI, includes AD Recycle Bin, PowerShell history viewer |
| DNS Manager | `dnsmgmt.msc` | Zones, records, forwarders |
| DHCP Manager | `dhcpmgmt.msc` | Scopes, leases, reservations |
| Group Policy Management Console | `gpmc.msc` | Domain-wide GPO creation/linking/reporting |

---

## 10. Command-Line Network & Service Discovery

All built in, no install required (run from `cmd` or PowerShell).

| Tool | Command | Purpose |
|---|---|---|
| nslookup | `nslookup host.name` | DNS lookups (A/AAAA/MX/TXT/NS), query a specific DNS server |
| Resolve-DnsName | `Resolve-DnsName host.name` (PowerShell) | Modern nslookup equivalent, structured object output |
| ping | `ping host` | ICMP reachability test |
| tracert | `tracert host` | Hop-by-hop route trace |
| pathping | `pathping host` | tracert + ping stats combined per hop |
| netstat | `netstat -ano` | Active connections/listening ports mapped to PID |
| Get-NetTCPConnection | `Get-NetTCPConnection` (PowerShell) | Structured netstat equivalent |
| arp | `arp -a` | ARP cache (MAC↔IP on local segment) |
| nbtstat | `nbtstat -a host` | NetBIOS name resolution/enumeration |
| net view | `net view \\host` / `net view /domain` | Enumerate SMB shares / domain machines |
| netsh | `netsh wlan show profiles`, `netsh interface ip show config`, `netsh advfirewall firewall show rule name=all`, `netsh trace start` | Swiss-army network config/diagnostics/packet capture |
| Test-NetConnection | `Test-NetConnection host -Port 443` (PowerShell) | Port-check + ping + route in one (netcat-lite) |
| Get-DnsClientCache | `Get-DnsClientCache` (PowerShell) | Inspect local DNS resolver cache |
| route | `route print` | Routing table |
| ipconfig | `ipconfig /all`, `/displaydns`, `/flushdns` | Adapter config, local DNS cache view/flush |
| whoami | `whoami /all` | Current user, SIDs, groups, privileges |
| systeminfo | `systeminfo` | Full host inventory: patches, hotfixes, domain, uptime |
| wmic / Get-CimInstance | `wmic service list brief` or `Get-CimInstance Win32_Service` | Query services, processes, installed software via WMI |
| sc | `sc query` | Service Control Manager query from CLI |

> No built-in equivalent of nmap for scanning a remote subnet's ports — `Test-NetConnection -Port X` only checks one host/port at a time. For real network scanning, external tools (e.g. nmap) are still needed.

---

## 11. Disk & Storage CLI Tools

| Tool | Command | Purpose |
|---|---|---|
| diskpart | `diskpart` (interactive) | Partitions, volumes, disk initialization, drive letters, MBR/GPT conversion, extend/shrink volumes |
| Storage PowerShell module | `Get-Disk`, `Get-Partition`, `Get-Volume`, `New-Partition`, `Resize-Partition` | Scriptable, modern equivalent of diskpart — preferred for automation |
| chkdsk | `chkdsk C: /f /r` | Filesystem integrity check/repair |
| fsutil | `fsutil fsinfo drives`, `fsutil quota query C:` | Low-level filesystem ops: quotas, reparse points, hardlinks, sparse files |
| format | `format E: /FS:NTFS` | Quick CLI volume format |
| mountvol | `mountvol` | Manage volume mount points |
| vssadmin | `vssadmin list shadows` | Volume Shadow Copy management (list/create/delete shadow copies — backups, forensics) |

---

## 12. Quick Reference — "I want to change X, where do I go?"

| Task | Tool |
|---|---|
| Rename PC / join domain | `sysdm.cpl` or `ms-settings:about` |
| Change static IP | `ncpa.cpl` → adapter properties, or `ms-settings:network-status` |
| Check what's using a port | `netstat -ano` + Task Manager Details tab (match PID) |
| See why an app crashed | `eventvwr.msc` → Windows Logs → Application |
| See failed logons | `eventvwr.msc` → Windows Logs → Security (Event ID 4625) |
| Manage local admin accounts | `lusrmgr.msc` (Pro+) or `netplwiz` |
| View/reset password policy (local) | `secpol.msc` → Account Policies |
| Import a cert for RDP/service | `mmc.exe` → Certificates (computer account) |
| Fix corrupted system files | `sfc /scannow` then `DISM /Online /Cleanup-Image /RestoreHealth` |
| Manage startup programs | `taskmgr` → Startup apps tab |
| Manage scheduled tasks | `taskschd.msc` |
| Firewall rule for an app/port | `wf.msc` |
| Manage shares | `fsmgmt.msc` or `net share` |
| Encrypt a drive | `control /name Microsoft.BitLockerDriveEncryption` |
| See installed drivers/rollback one | `devmgmt.msc` |
| Manage Wi-Fi profiles | `netsh wlan show profiles` (cmd), or Settings → Network & internet → Wi-Fi → Manage known networks |

---

**Note on Windows 11 editions:** `gpedit.msc`, `secpol.msc`, `lusrmgr.msc`, `printmanagement.msc`, and full BitLocker are Pro/Enterprise/Education only — Home edition hides or removes them (workarounds exist but aren't officially supported).
