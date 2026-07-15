# PowerShell Remoting — IR Investigation Guide

## Overview
PowerShell Remoting uses WinRM over port 5985 (HTTP) or 5986 (HTTPS).
Enables remote investigation across the entire Windows estate from a single
investigation workstation without RDP into each machine individually.

## Enable WinRM (run as Administrator on target)
```powershell
Enable-PSRemoting -Force
```

## Interactive Session (single machine)
```powershell
Enter-PSSession -ComputerName W02
exit
```

## Run Command Across Multiple Machines
```powershell
Invoke-Command -ComputerName W01,W02,W03,DC01 -ScriptBlock { hostname }
```

## Lab 1 Results — Environment Baseline
| Machine | Hostname | Time | Timezone |
|---------|----------|------|----------|
| W01 | W01 | 2026-07-15 05:10:02 | UTC |
| W02 | W02 | 2026-07-15 05:10:02 | UTC |
| W03 | W03 | 2026-07-15 05:10:02 | UTC |
| DC01 | DC01 | 2026-07-15 05:10:03 | UTC |

## IR Notes
- All machines UTC — timestamps are directly comparable, no conversion needed
- Establish hostname/time baseline at start of every investigation
- Time sync confirms no clock manipulation on endpoints
