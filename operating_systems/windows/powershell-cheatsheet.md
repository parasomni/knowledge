# PowerShell — Comprehensive Cheatsheet

Covers PowerShell 5.1 (Windows built-in) and PowerShell 7+ (`pwsh`, cross-platform). Cmdlets follow a strict `Verb-Noun` naming convention — once you know the verbs, you can often guess the cmdlet.

---

## 1. Core Concepts

- **Everything is an object.** Cmdlets output .NET objects, not text — output flows through the pipeline (`|`) as structured objects with properties/methods, not raw strings like in bash.
- **Cmdlet naming:** `Verb-Noun` (e.g. `Get-Process`, `Stop-Service`). Standard verbs: `Get`, `Set`, `New`, `Remove`, `Start`, `Stop`, `Restart`, `Enable`, `Disable`, `Add`, `Clear`, `Copy`, `Move`, `Rename`, `Test`, `Invoke`, `Import`, `Export`, `ConvertTo`/`ConvertFrom`. Full list: `Get-Verb`.
- **Providers** expose non-filesystem data stores as navigable "drives": `Get-PSDrive` shows them (`C:`, `HKLM:`, `HKCU:`, `Env:`, `Cert:`, `Function:`, `Alias:`, `Variable:`). You can `cd HKLM:\SOFTWARE` like a folder.
- **Case-insensitive** by default (unlike bash).
- **Execution Policy** governs script execution: `Get-ExecutionPolicy`, `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser` (common dev setting; `Restricted` is the secure default, `Bypass`/`Unrestricted` disable protection entirely).

---

## 2. Getting Help (start here for anything)

| Command | Purpose |
|---|---|
| `Get-Help <cmdlet>` | Show help for a cmdlet |
| `Get-Help <cmdlet> -Examples` | Just usage examples |
| `Get-Help <cmdlet> -Full` | Full help incl. parameter details |
| `Get-Help <cmdlet> -Online` | Opens the Microsoft Learn page |
| `Update-Help` | Downloads latest help content (run once, elevated) |
| `Get-Command` | List all available commands |
| `Get-Command -Verb Get -Noun *service*` | Search commands by verb/noun pattern |
| `Get-Command -Module <name>` | List cmdlets in a specific module |
| `Get-Member` (alias `gm`) | List properties/methods of a piped object — **the most-used discovery command** |
| `Get-Verb` | List all approved PowerShell verbs |

```powershell
Get-Process | Get-Member                  # what properties/methods does a process object have?
Get-Help Get-ChildItem -Examples
```

---

## 3. The Pipeline & Object Filtering

| Cmdlet | Alias | Purpose |
|---|---|---|
| `Where-Object` | `?`, `where` | Filter objects by condition |
| `Select-Object` | `select` | Pick specific properties / first-N / unique |
| `Sort-Object` | `sort` | Sort by property |
| `Group-Object` | `group` | Group by property value |
| `Measure-Object` | `measure` | Count/sum/average/min/max |
| `ForEach-Object` | `%`, `foreach` | Run a script block per pipeline object |
| `Tee-Object` | `tee` | Split pipeline output to a file/variable and continue |
| `Compare-Object` | `diff` | Diff two object collections |

```powershell
Get-Process | Where-Object { $_.CPU -gt 50 } | Sort-Object CPU -Descending | Select-Object -First 5
Get-Service | Group-Object Status
Get-ChildItem C:\Logs | Measure-Object -Property Length -Sum
```

**Comparison operators (not `<`/`>`/`==`):** `-eq -ne -gt -ge -lt -le -like -notlike -match -notmatch -contains -notcontains -in -notin`. String comparisons are case-*insensitive* by default; prefix with `c` for case-sensitive (`-ceq`, `-cmatch`, etc.).

**Logical operators:** `-and -or -not -xor` (not `&&`/`||` — those exist too in PS7+ as pipeline chain operators, different meaning).

---

## 4. Formatting & Output

| Cmdlet | Purpose |
|---|---|
| `Format-Table` (`ft`) | Table view (often the default for object collections) |
| `Format-List` (`fl`) | List view — good for objects with many properties |
| `Format-Wide` (`fw`) | Single-column wide view |
| `Out-GridView` | Interactive filterable GUI grid (PS7 needs `Microsoft.PowerShell.GraphicalTools` module) |
| `Out-File` | Write output to a file |
| `Out-String` | Convert objects to a single string |
| `Out-Null` | Discard output |
| `ConvertTo-Json` / `ConvertFrom-Json` | JSON serialize/deserialize |
| `ConvertTo-Csv` / `ConvertFrom-Csv` / `Export-Csv` / `Import-Csv` | CSV handling |
| `ConvertTo-Html` | Render objects as an HTML table |
| `ConvertTo-SecureString` | Build a SecureString (passwords, tokens) |

```powershell
Get-Process | Export-Csv procs.csv -NoTypeInformation
Get-ADUser -Filter * | ConvertTo-Json | Out-File users.json
```

---

## 5. Variables, Types, Operators

```powershell
$x = 5                        # variable, no type declaration needed
[int]$y = "10"                # explicit type cast
$arr = @(1,2,3)                # array
$hash = @{ Name = "Bob"; Age = 30 }   # hashtable
$obj = [PSCustomObject]@{ Name = "Bob"; Age = 30 }  # ad-hoc object

$env:PATH                      # environment variable
$PSVersionTable                # PowerShell/OS/CLR version info
$_ / $PSItem                   # current pipeline object inside a script block
$?                              # did the last command succeed
$LASTEXITCODE                   # exit code of last native (.exe) command
$Error                          # array of recent errors, $Error[0] = most recent
```

**Arithmetic:** `+ - * / %`. **Assignment:** `= += -= *= /= ++ --`. **String concat:** `+` or interpolation `"Hello $name"` / `"Sum: $($x+1)"`.

---

## 6. Control Flow

```powershell
if ($x -gt 5) { "big" } elseif ($x -eq 5) { "equal" } else { "small" }

foreach ($item in $collection) { $item }
for ($i=0; $i -lt 10; $i++) { $i }
while ($x -lt 10) { $x++ }
do { $x++ } while ($x -lt 10)

switch ($val) {
    1       { "one" }
    {$_ -gt 5} { "big" }   # script block condition
    default { "other" }
}

try { Risky-Command } catch { "Error: $($_.Exception.Message)" } finally { "cleanup" }
```

---

## 7. Functions & Scripts

```powershell
function Get-Square {
    param(
        [Parameter(Mandatory)][int]$Number
    )
    return $Number * $Number
}
Get-Square -Number 5

# Advanced function with pipeline input, like a real cmdlet
function Test-Something {
    [CmdletBinding()]
    param(
        [Parameter(ValueFromPipeline)]$InputObject
    )
    process { "Got: $InputObject" }
}
```

- `.ps1` = script, `.psm1` = module, `.psd1` = module manifest (metadata)
- Dot-source a script to load its functions into the current session: `. .\script.ps1`
- Profile scripts (auto-run on shell start): `$PROFILE` shows the path; `notepad $PROFILE` to edit

---

## 8. Modules

| Command | Purpose |
|---|---|
| `Get-Module` | Modules loaded in current session |
| `Get-Module -ListAvailable` | All modules installed on the system |
| `Import-Module <name>` | Load a module |
| `Remove-Module <name>` | Unload a module |
| `Find-Module <name>` | Search the PowerShell Gallery |
| `Install-Module <name> -Scope CurrentUser` | Install from PowerShell Gallery |
| `Update-Module` | Update installed modules |
| `Get-InstalledModule` | List modules installed via Install-Module |

Common built-in/RSAT modules: `ActiveDirectory`, `DnsServer`, `DhcpServer`, `GroupPolicy`, `Microsoft.PowerShell.Management`, `Microsoft.PowerShell.Security`, `Storage`, `NetTCPIP`, `NetSecurity` (firewall), `Hyper-V`, `PKI` (certificates), `ScheduledTasks`.

---

## 9. Filesystem & Registry (Provider-based)

Filesystem cmdlets work identically against the registry, since both are PSDrive providers.

| Cmdlet | Alias | Purpose |
|---|---|---|
| `Get-ChildItem` | `ls`, `dir`, `gci` | List items (files, or registry subkeys) |
| `Get-Item` | `gi` | Get a single item |
| `Get-ItemProperty` | `gp` | Get properties/values (file attributes, or registry values) |
| `Set-ItemProperty` | `sp` | Set a property/value |
| `New-Item` | `ni` | Create file/folder/registry key |
| `Remove-Item` | `rm`, `del`, `ri` | Delete file/folder/registry key |
| `Copy-Item` | `cp`, `copy` | Copy |
| `Move-Item` | `mv`, `move` | Move/rename |
| `Rename-Item` | `ren` | Rename |
| `Test-Path` | | Check existence |
| `Set-Location` | `cd`, `sl` | Change directory/drive |
| `Get-Content` | `cat`, `gc`, `type` | Read file content |
| `Set-Content` / `Add-Content` | `sc`/`ac` | Write/append file content |
| `Get-Acl` / `Set-Acl` | | Read/write NTFS or registry permissions |

```powershell
Get-ChildItem HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
New-ItemProperty -Path HKCU:\Software\Test -Name "Setting" -Value 1 -PropertyType DWord
Get-ChildItem -Path C:\ -Recurse -Filter *.log -ErrorAction SilentlyContinue
```

---

## 10. Processes, Services, System

| Cmdlet | Purpose |
|---|---|
| `Get-Process` (`ps`) | List/query running processes |
| `Stop-Process` (`kill`) | Terminate a process |
| `Start-Process` | Launch a process (supports `-Verb RunAs` for elevation) |
| `Wait-Process` | Block until a process exits |
| `Get-Service` | List/query services |
| `Start-Service` / `Stop-Service` / `Restart-Service` | Control service state |
| `Set-Service` | Change service startup type/config |
| `New-Service` | Create a Windows service |
| `Get-CimInstance` (replaces `Get-WmiObject`) | Query WMI/CIM classes (`Win32_OperatingSystem`, `Win32_BIOS`, `Win32_LogicalDisk`, etc.) |
| `Get-HotFix` | Installed updates/patches |
| `Get-ComputerInfo` | Consolidated OS/hardware info |
| `Restart-Computer` / `Stop-Computer` | Reboot/shutdown (local or `-ComputerName` remote) |
| `Get-EventLog` (legacy) / `Get-WinEvent` (modern) | Query event logs |
| `Get-ScheduledTask` / `Register-ScheduledTask` / `Start-ScheduledTask` | Task Scheduler management |
| `Get-Counter` | Live performance counter data |

```powershell
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, Version, LastBootUpTime
Get-WinEvent -LogName Security -MaxEvents 50 | Where-Object Id -eq 4625
Start-Process powershell -Verb RunAs   # UAC-elevate a new shell
```

---

## 11. Networking

| Cmdlet | Purpose |
|---|---|
| `Get-NetIPAddress` | Configured IP addresses per interface |
| `Get-NetIPConfiguration` | ipconfig-style summary |
| `New-NetIPAddress` / `Set-NetIPAddress` | Assign static IP |
| `Get-NetAdapter` | List network adapters |
| `Enable-NetAdapter` / `Disable-NetAdapter` | Toggle adapter |
| `Get-NetRoute` | Routing table |
| `Get-NetTCPConnection` | netstat equivalent, structured |
| `Get-NetFirewallRule` / `New-NetFirewallRule` | Windows Firewall rule management |
| `Test-NetConnection` | Ping + port-check + traceroute combined |
| `Resolve-DnsName` | DNS lookups |
| `Get-DnsClientServerAddress` | Configured DNS servers |
| `Get-SmbShare` / `New-SmbShare` | SMB share management |
| `Get-SmbConnection` | Active SMB sessions |

```powershell
Test-NetConnection -ComputerName dc01.corp.local -Port 389
New-NetFirewallRule -DisplayName "Allow RDP" -Direction Inbound -Protocol TCP -LocalPort 3389 -Action Allow
```

---

## 12. Security & Credentials

| Cmdlet | Purpose |
|---|---|
| `Get-Credential` | Prompt for and store a credential object |
| `ConvertTo-SecureString` / `ConvertFrom-SecureString` | Handle secure strings (passwords) |
| `Get-ExecutionPolicy` / `Set-ExecutionPolicy` | Script execution restrictions |
| `Get-Acl` / `Set-Acl` | File/registry permissions |
| `Get-LocalUser` / `New-LocalUser` / `Set-LocalUser` | Local account management |
| `Get-LocalGroup` / `Get-LocalGroupMember` / `Add-LocalGroupMember` | Local group management |
| `Get-AuthenticodeSignature` | Check a script/binary's digital signature |
| `Get-PfxCertificate` | Inspect a certificate file |
| `New-SelfSignedCertificate` | Create a self-signed cert (PKI module) |
| `Unblock-File` | Remove the "downloaded from internet" mark-of-the-web flag |

```powershell
$cred = Get-Credential
Invoke-Command -ComputerName srv01 -Credential $cred -ScriptBlock { Get-Service }
```

---

## 13. Remoting & Remote Management

| Cmdlet | Purpose |
|---|---|
| `Enable-PSRemoting` | Enable WinRM-based remoting on this machine (elevated) |
| `Test-WSMan` | Check if WinRM is reachable on a target |
| `Enter-PSSession -ComputerName <host>` | Interactive remote shell |
| `Invoke-Command -ComputerName <host> -ScriptBlock {...}` | Run a script block remotely (one-shot or against many hosts) |
| `New-PSSession` / `Get-PSSession` / `Remove-PSSession` | Persistent remote sessions (reusable, stateful) |
| `New-CimSession` | Remote CIM/WMI session (works even where WinRM is off, uses DCOM fallback) |

```powershell
Invoke-Command -ComputerName srv01,srv02 -ScriptBlock { Get-Service spooler }
Enter-PSSession -ComputerName dc01
```

---

## 14. Active Directory (RSAT `ActiveDirectory` module)

| Cmdlet | Purpose |
|---|---|
| `Get-ADUser` / `New-ADUser` / `Set-ADUser` / `Remove-ADUser` | User object management |
| `Get-ADGroup` / `Get-ADGroupMember` / `Add-ADGroupMember` | Group management |
| `Get-ADComputer` | Computer objects |
| `Get-ADOrganizationalUnit` / `New-ADOrganizationalUnit` | OU management |
| `Get-ADDomain` / `Get-ADForest` | Domain/forest info |
| `Get-ADDomainController` | List DCs |
| `Search-ADAccount -AccountInactive` | Find stale accounts |
| `Set-ADAccountPassword` / `Unlock-ADAccount` | Password/lockout management |
| `Get-ADReplicationFailure` | Replication health |
| `Get-GPO` / `New-GPLink` (GroupPolicy module) | GPO management |
| `Get-DnsServerZone` / `Get-DhcpServerv4Scope` | DNS/DHCP (own modules) |

---

## 15. Package & Update Management

| Cmdlet | Purpose |
|---|---|
| `Get-WindowsCapability -Online` | List optional features (incl. RSAT) |
| `Add-WindowsCapability -Online -Name <name>` | Install an optional feature |
| `Get-WindowsFeature` / `Install-WindowsFeature` | Server roles/features (Windows Server) |
| `winget` (separate CLI, not a cmdlet) | Modern app package manager (`winget install`, `winget upgrade --all`) |
| `Get-Package` / `Install-Package` | PackageManagement (OneGet) — generic provider-based package installs |
| `Get-WUJob` / `Install-WindowsUpdate` (PSWindowsUpdate module, not built-in) | Windows Update automation |

---

## 16. Jobs & Background/Parallel Execution

| Cmdlet | Purpose |
|---|---|
| `Start-Job` | Run a script block as a background job |
| `Get-Job` / `Receive-Job` / `Stop-Job` / `Remove-Job` | Manage/collect job results |
| `Wait-Job` | Block until job(s) complete |
| `Start-ThreadJob` (PS7, `ThreadJob` module) | Lighter-weight threaded jobs |
| `ForEach-Object -Parallel` (PS7+) | Parallel pipeline execution |

```powershell
1..5 | ForEach-Object -Parallel { Test-Connection "host$_" -Count 1 } -ThrottleLimit 5
```

---

## 17. Useful One-Liners

```powershell
# Top 5 memory-hungry processes
Get-Process | Sort-Object WS -Descending | Select-Object -First 5 Name, WS

# Find files modified in the last 24h
Get-ChildItem C:\ -Recurse -ErrorAction SilentlyContinue | Where-Object { $_.LastWriteTime -gt (Get-Date).AddDays(-1) }

# Local admins group members
Get-LocalGroupMember Administrators

# All listening ports with owning process name
Get-NetTCPConnection -State Listen | Select-Object LocalAddress, LocalPort, @{N='Process';E={(Get-Process -Id $_.OwningProcess).Name}}

# Recently failed logons
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 20

# Export installed software list
Get-CimInstance Win32_Product | Select-Object Name, Version | Export-Csv software.csv -NoTypeInformation

# Bulk rename files
Get-ChildItem *.txt | Rename-Item -NewName { $_.Name -replace 'old','new' }

# Check if running elevated
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
```

---

## 18. Aliases Worth Knowing (bash/cmd habits that work)

| Alias | Real Cmdlet |
|---|---|
| `ls`, `dir` | `Get-ChildItem` |
| `cd` | `Set-Location` |
| `pwd` | `Get-Location` |
| `cat`, `type` | `Get-Content` |
| `cp`, `copy` | `Copy-Item` |
| `mv`, `move` | `Move-Item` |
| `rm`, `del` | `Remove-Item` |
| `ps` | `Get-Process` |
| `kill` | `Stop-Process` |
| `curl`, `wget` | `Invoke-WebRequest` (aliased — real curl/wget also exist separately in modern Windows) |
| `man` | `Get-Help` |
| `cls`, `clear` | `Clear-Host` |
| `history` | `Get-History` |
| `echo` | `Write-Output` |

`Get-Alias` lists all current aliases; `New-Alias` defines your own.

---

## 19. Notes on PowerShell 7+ (`pwsh`) vs Windows PowerShell 5.1

- PS7 is cross-platform (Windows/Linux/macOS), open source, side-by-side installable with 5.1.
- Some Windows-only modules (`ActiveDirectory`, older WMI-heavy modules) still require 5.1 or the compatibility layer (`Import-Module -UseWindowsPowerShell`).
- PS7 adds: `ForEach-Object -Parallel`, ternary operator (`$x ? "a" : "b"`), pipeline chain operators (`&&`, `||`), null-coalescing (`??`, `??=`), better error views.
- `$PSVersionTable.PSVersion` tells you which one you're in.
