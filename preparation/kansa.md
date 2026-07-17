# Kansa — Remote Windows Investigation

## What Kansa Is

Kansa is a PowerShell-based IR framework that collects evidence across multiple Windows machines simultaneously using WinRM. It does not detect malware — it collects evidence for analyst review.

```
One analyst on IR_windows
        │
        ▼
      Kansa
        │
 ┌──────┼──────┐
 │      │      │
W01    W02    DC01
 │      │      │
Collect evidence simultaneously
        │
        ▼
Results saved as CSV files
```

---

## Location in Lab

```
C:\Users\ir_user\Desktop\Tools\Kansa
```

---

## Setup

```powershell
cd C:\Users\ir_user\Desktop\Tools\Kansa
ls -r *.ps1 | Unblock-File
Set-ExecutionPolicy Unrestricted -Force
"W01","W02","W03","DC01" | Out-File -FilePath .\hostlist.txt
cat .\hostlist.txt
```

**What each command does:**

| Command | Purpose |
|---------|---------|
| `Unblock-File` | Removes internet download block from all scripts |
| `Set-ExecutionPolicy Unrestricted` | Allows PowerShell to run scripts |
| `Out-File` | Creates the target machine list |

---

## How Kansa Works

Kansa needs three things before it can run:

### 1 — hostlist.txt

Tells Kansa which machines to investigate:

```
W01
W02
W03
DC01
```

### 2 — Modules

PowerShell scripts that each perform one investigation task:

```
Kansa\Modules\
├── Log\
│     Get-LogWinEvent.ps1      ← collect Windows Event Logs
├── Disk\
│     Get-TempDirListing.ps1   ← list Temp folder contents
│     Get-LocalUsers.ps1       ← list local user accounts
├── Net\
│     Get-Netstat.ps1          ← active network connections
├── ASEP\
│     Get-Autorunsc.ps1        ← persistence mechanisms
```

### 3 — Modules.conf

The shopping list — controls which modules run. Lines starting with `#` are disabled.

```
# Log\Get-LogWinEvent.ps1 Security    ← disabled
Log\Get-LogWinEvent.ps1 Security      ← enabled
```

---

## Exercise 1 — Configure Modules

```powershell
notepad .\Modules\Modules.conf
```

Remove `#` from these three lines:

```
Log\Get-LogWinEvent.ps1 Security
Disk\Get-TempDirListing.ps1
Config\Get-LocalUsers.ps1
```

**What each module reveals during IR:**

| Module | Investigation Question |
|--------|----------------------|
| `Get-LogWinEvent Security` | Who logged in, when, and from where? |
| `Get-TempDirListing` | Is malware staged in Temp folders? |
| `Get-LocalUsers` | Were backdoor accounts created? |

---

## Exercise 2 — Run Kansa

```powershell
.\kansa.ps1 -TargetList .\hostlist.txt -ModulePath .\Modules -Verbose
```

**Command breakdown:**

| Part | Purpose |
|------|---------|
| `-TargetList .\hostlist.txt` | Which machines to investigate |
| `-ModulePath .\Modules` | Run enabled modules from Modules.conf |
| `-Verbose` | Show progress as it runs |

**Output structure:**

```
Output_20260717121522\
    DNSCache\
        W01-DNSCache.csv
        W02-DNSCache.csv
        W03-DNSCache.csv
        DC01-DNSCache.csv
    Netstat\
        W01-Netstat.csv
        W02-Netstat.csv
    PrefetchListing\
    ProcsWMI\
    LogUserAssist\
    WMIRecentApps\
```

One folder per module. One CSV per machine. Open in Excel or analyze with PowerShell.

---

## Exercise 3 — Autoruns Module

Autoruns is a Sysinternals tool that finds every persistence mechanism on a Windows machine with SHA256 hashes.

### Setup — Copy Binary to Kansa

```powershell
copy C:\Users\ir_user\Desktop\Tools\Autoruns\autorunsc.exe .\Modules\bin\
ls .\Modules\bin\
```

### Run with -Pushbin

```powershell
.\kansa.ps1 -TargetList .\hostlist.txt -ModulePath ".\Modules\ASEP\Get-Autorunsc.ps1" -Pushbin
```

**What `-Pushbin` does:**

```
IR_windows has autorunsc.exe in Kansa\Modules\bin\
        │
        ▼
Kansa copies autorunsc.exe to W01 via WinRM
        │
Autorunsc runs on W01
Finds every persistence mechanism
Records SHA256 hash of each binary
        │
Results returned as CSV to IR_windows
        │
autorunsc.exe removed from W01
```

### Output

```
Output_20260717124152\
    Autorunsc\
        DC01-Autorunsc.csv   (1.5 MB)
        W01-Autorunsc.csv    (1.5 MB)
        W02-Autorunsc.csv    (1.5 MB)
        W03-Autorunsc.csv    (1.5 MB)
```

### Sample Evidence From W01

```
Time           : 8/5/1978 6:25 PM
Entry Location : HKLM\System\CurrentControlSet\Control\Session Manager\BootExecute
Entry          : autocheck autochk /q /v *
Enabled        : enabled
Category       : Boot Execute
Description    : Auto Check Utility
Signer         : (Verified) Microsoft Windows
Company        : Microsoft Corporation
Image Path     : c:\windows\system32\autochk.exe
Version        : 10.0.20348.1
MD5            : 60D35DCBD7F61CED00F96ACA79B5F63A
SHA-256        : 25ED9D247D1849B6B9B403C93C3966C180E645726D0EDC706C94F56450EB7FD2
PSComputerName : W01

Time           : 5/8/1956 12:02 AM
Entry Location : HKLM\SOFTWARE\Classes\Htmlfile\Shell\Open\Command\(Default)
Entry          : C:\Program Files\Internet Explorer\iexplore.exe
Enabled        : enabled
Category       : Hijacks
Description    : Internet Explorer
Signer         : (Verified) Microsoft Corporation
Company        : Microsoft Corporation
Image Path     : c:\program files\internet explorer\iexplore.exe
Version        : 11.0.20348.380
MD5            : 9B72DCA2D2080B95EF233F30026D206D
SHA-256        : 5FDFC590B1F84FBC06FDA7BCF15B43D202E543CEB3664E176CBA98D3B87FFB89
PSComputerName : W01
```

### Persistence Categories Collected

| Category | What It Covers |
|----------|---------------|
| Boot Execute | Runs before Windows fully loads |
| Hijacks | Browser and file handler hijacking |
| Services | Windows services |
| Scheduled Tasks | Scheduled execution |
| Startup | Startup folder entries |
| Registry | Run key persistence |

### How to Use the Hashes During IR

```powershell
# Extract all hashes from W01 for VirusTotal lookup
Import-Csv Output_20260717124152\Autorunsc\W01-Autorunsc.csv |
    Where-Object { $_.'SHA-256' -ne '' } |
    Select-Object Entry, 'Image Path', 'SHA-256', Signer |
    Export-Csv W01-hashes-for-lookup.csv -NoTypeInformation
```

Submit SHA256 hashes to VirusTotal:
- **Verified Microsoft** = clean
- **Unsigned** = investigate
- **Flagged by vendors** = malicious

---

## IR Value Summary

| Module | What It Reveals |
|--------|----------------|
| `Get-LogWinEvent Security` | Authentication, privilege escalation, process creation |
| `Get-TempDirListing` | Malware staged before execution |
| `Get-LocalUsers` | Backdoor accounts created by attacker |
| `Get-Autorunsc` | Every persistence mechanism with SHA256 hashes |

---

## Key Lesson

Kansa does not decide if a machine is compromised. It collects evidence. You decide what it means.

The analyst correlates findings across machines:

```
Security log shows logon at 3 AM on W01
        │
        ▼
Temp folder on W01 has unknown .ps1 file created at 3:01 AM
        │
        ▼
Autoruns shows that .ps1 added as scheduled task at 3:02 AM
        │
        ▼
New local user account created at 3:05 AM
        │
        ▼
That is the attack timeline
```

That correlation tells the story of what happened.
