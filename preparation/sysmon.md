# Sysmon — Establish More Visibility

## What Sysmon Is
Sysmon (System Monitor) is a Windows system service that logs detailed
system activity to the Windows Event Log. It provides granular visibility
that default Windows logging cannot match.

## Key Event IDs

| ID | Event | Security Value |
|----|-------|---------------|
| 1 | Process creation | Full cmdline, hash, parent process |
| 3 | Network connection | Process + destination IP/port |
| 6 | Driver loaded | BYOVD detection |
| 7 | DLL loaded | DLL injection detection |
| 8 | CreateRemoteThread | Process injection |
| 10 | Process accessed | LSASS dumping detection |
| 11 | File created | Payload dropped |
| 12/13 | Registry modified | Persistence detection |
| 22 | DNS query | C2 domain detection |
| 23 | File deleted | Evidence destruction |
| 25 | Process tampering | Process hollowing |

## Installation

### Download
- Sysmon: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
- Config: https://github.com/SwiftOnSecurity/sysmon-config

### Install Locally
```powershell
.\Sysmon64.exe -i -accepteula
.\Sysmon64.exe -c sysmonconfig.xml
```

### Deploy Remotely via PowerShell Remoting
```powershell
# Create sessions
$session2 = New-PSSession -ComputerName 192.168.238.144 -Credential $cred
$session3 = New-PSSession -ComputerName 192.168.238.145 -Credential $cred

# Copy Sysmon to remote machines
Copy-Item -Path "C:\Tools\Sysmon" -Destination "C:\Temp\Sysmon" -Recurse -ToSession $session2 -Force
Copy-Item -Path "C:\Tools\Sysmon" -Destination "C:\Temp\Sysmon" -Recurse -ToSession $session3 -Force

# Install and configure
Invoke-Command -Session $session2,$session3 -ScriptBlock {
    cd C:\Temp\Sysmon
    .\Sysmon64.exe -i -accepteula
    .\Sysmon64.exe -c sysmonconfig.xml
}
```

### Verify Running
```powershell
Invoke-Command -ComputerName 192.168.238.144,192.168.238.145 -Credential $cred -ScriptBlock {
    Get-Service Sysmon64 | Select-Object Name, Status
}
```

## Query Sysmon Events
```powershell
# All recent events
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 50

# Process creation only
Get-WinEvent -FilterHashtable @{LogName="Microsoft-Windows-Sysmon/Operational"; Id=1} -MaxEvents 20

# Network connections
Get-WinEvent -FilterHashtable @{LogName="Microsoft-Windows-Sysmon/Operational"; Id=3} -MaxEvents 20

# LSASS access attempts
Get-WinEvent -FilterHashtable @{LogName="Microsoft-Windows-Sysmon/Operational"; Id=10} |
    Where-Object { $_.Message -match "lsass" }
```

## Lab Results
| Machine | IP | Status | Events Generating |
|---------|-----|--------|------------------|
| Client01 | 192.168.238.142 | Running ✅ | Yes |
| W02 | 192.168.238.144 | Running ✅ | Yes |
| W03 | 192.168.238.145 | Running ✅ | Yes |

## Sysmon vs Default Windows Logging

Default Event 4688 (process creation):
- Process name only

Sysmon Event 1 (process creation):
- Full command line with arguments
- SHA256 hash of the binary
- Parent process name and command line
- User and logon session
- File creation time

The difference is the difference between knowing powershell.exe ran
and knowing powershell.exe ran an encoded command spawned by winword.exe
after a user opened a malicious document.
