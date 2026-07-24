# Windows Remote Evidence Acquisition

## Overview

After anomaly detection identifies suspicious machines, the next step is
collecting forensic evidence remotely. This covers three acquisition types:
disk artifacts, RAM, and memory analysis.

---

## Evidence Acquisition Order

```
Incident confirmed on target machine
        │
        ▼
1. RAM dump FIRST — disappears on reboot
        │
        ▼
2. Network state — active connections drop quickly
        │
        ▼
3. Running processes — snapshot before malware detects IR
        │
        ▼
4. Disk artifacts — KAPE or CyLR
        │
        ▼
5. Full disk image — only if needed
```

Never reboot before acquiring RAM.

---

## Tool 1 — KAPE (Kroll Artifact Parser and Extractor)

### What It Is
Modular evidence collection tool. You choose which artifacts to collect
using predefined target lists. Output can be a folder or VHDX disk image.

### Location in Lab
```
C:\Users\ir_user\Desktop\Tools\KAPE\
```

### Three Required Parameters

| Parameter | Purpose | Example |
|-----------|---------|---------|
| `--tsource` | Source volume to collect from | `C:\` |
| `--target` | Which artifacts to collect | `!BasicCollection` |
| `--tdest` | Where to save results | `E:\result` |

### Run Locally
```powershell
.\kape.exe --tsource C:\ --target !BasicCollection --tdest E:\result --vhdx local
```

`--vhdx local` packages everything into a single mountable VHDX file.

### Run Remotely via PowerShell Remoting
```powershell
# Copy KAPE to remote machine
Copy-Item -Path "C:\Tools\KAPE" -Destination "\\W02\C$\Temp\KAPE" -Recurse

# Execute KAPE remotely
Invoke-Command -ComputerName W02 -Credential $cred -ScriptBlock {
    cd C:\Temp\KAPE
    .\kape.exe --tsource C:\ --target !BasicCollection --tdest C:\Temp\results --vhdx W02
}

# Copy results back to investigation machine
Copy-Item "\\W02\C$\Temp\results\W02.vhdx" -Destination "C:\IR\W02.vhdx"
```

### What BasicCollection Gathers

| Artifact | Forensic Value |
|----------|--------------|
| Event Logs | Authentication, process creation, service install |
| Registry Hives | Persistence, user activity, system config |
| Prefetch | Execution history — what ran and when |
| LNK Files | Recently opened files |
| Scheduled Tasks | Persistence mechanisms |
| Amcache | SHA1 hashes of executed binaries |
| Shellbags | Folder navigation history |
| Browser History | Web activity |

---

## Tool 2 — CyLR

### What It Is
Zero-configuration evidence collection. No targets to choose —
runs automatically and collects the most important artifacts.
Best for large scale collection across many machines.

### KAPE vs CyLR

| Feature | KAPE | CyLR |
|---------|------|-------|
| Configuration | Required | None |
| Output | Folder or VHDX | ZIP archive |
| Customization | High | Limited |
| Best for | Targeted collection | Quick at-scale collection |
| Command | Complex | Simple |

### Run Locally
```powershell
.\CyLR.exe -od C:\results
```

### Run Across Multiple Machines
```powershell
$machines = @("W01","W02","W03","DC01")

Invoke-Command -ComputerName $machines -Credential $cred -ScriptBlock {
    cd C:\Temp
    .\CyLR.exe -od C:\Temp\results
}
```

One command — runs simultaneously on all machines simultaneously.

---

## Tool 3 — RAM Dump (Native PowerShell)

### Why RAM Matters

RAM contains what disk cannot show:
- Malware running entirely in memory with no file on disk
- Decrypted credentials in LSASS
- C2 communication strings
- Encryption keys
- Process memory with attacker commands

### Native PowerShell Method (No Third Party Tools)

```powershell
$ss = Get-CimInstance -ClassName MSFT_StorageSubSystem `
      -Namespace Root\Microsoft\Windows\Storage

Invoke-CimMethod -InputObject $ss `
    -MethodName "GetDiagnosticInfo" `
    -Arguments @{
        DestinationPath="C:\IR\dmp"
        IncludeLiveDump=$true
    }
```

Verify the dump:
```powershell
ls C:\IR\dmp\localhost\LiveDump.dmp
```

### Run Remotely
```powershell
Invoke-Command -ComputerName W02 -Credential $cred -ScriptBlock {
    $ss = Get-CimInstance -ClassName MSFT_StorageSubSystem `
          -Namespace Root\Microsoft\Windows\Storage
    Invoke-CimMethod -InputObject $ss `
          -MethodName "GetDiagnosticInfo" `
          -Arguments @{DestinationPath="C:\IR\dmp"; IncludeLiveDump=$true}
}

# Copy dump back
Copy-Item "\\W02\C$\IR\dmp\localhost\LiveDump.dmp" -Destination "C:\IR\W02-memory.dmp"
```

---

## Tool 4 — MemProcFS (Mount Memory Images)

### What It Is

MemProcFS mounts a memory dump as a virtual filesystem.
Browse forensic artifacts like a normal folder — no plugin commands needed.

### vs Volatility

| Feature | Volatility | MemProcFS |
|---------|-----------|-----------|
| Interface | CLI plugins | Virtual filesystem |
| Automation | Manual — one plugin at a time | Automatic — runs everything |
| Timeline | Manual correlation | Auto-generated |
| Best for | Deep targeted analysis | Quick comprehensive overview |

Use both — MemProcFS for initial overview, Volatility for deep investigation.

### Location in Lab
```
C:\Users\ir_user\Desktop\Tools\MemProcFS\
```

### Basic Mode — Browse Memory
```powershell
.\MemProcFS.exe -f C:\IR\dmp\localhost\LiveDump.dmp
```

A new drive mounts (e.g. M:\) with this structure:
```
M:\
├── name\       ← processes by name
├── pid\        ← processes by PID
│     └── 1234\
│           ├── files\    ← open files
│           ├── modules\  ← loaded DLLs
│           └── net\      ← network connections
├── sys\
│     └── net\  ← all network connections
└── forensic\   ← auto-generated reports
      ├── timeline\
      ├── yara\
      └── csv\
```

### Forensic Mode — Automatic Analysis
```powershell
.\MemProcFS.exe -f C:\IR\dmp\localhost\LiveDump.dmp -forensic 1
```

Automatically generates:
- Complete process timeline
- Network connection history
- Registry analysis
- YARA malware scanning
- Evidence of execution

### Analyse with Volatility After MemProcFS Overview
```bash
vol -f LiveDump.dmp windows.pslist     # running processes
vol -f LiveDump.dmp windows.netscan    # network connections
vol -f LiveDump.dmp windows.malfind    # injected code
vol -f LiveDump.dmp windows.hashdump   # password hashes
```

---

## Decision Tree

```
Need quick collection across many machines?
        YES → CyLR

Need specific targeted artifacts?
        YES → KAPE with specific target

Need RAM content?
        YES → PowerShell CimInstance method

Need to analyse RAM?
        YES → MemProcFS for overview, Volatility for deep dive
```
