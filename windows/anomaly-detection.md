# Windows Anomaly Detection

## Overview

Remote anomaly detection across Windows endpoints using PowerShell Remoting.
No agents required — uses built-in Windows administration tools only.

This is the first phase of active investigation — casting a wide net across all
machines to find anomalies before doing deep-dive evidence collection.

```
Cast wide net across all machines
        │
        ▼
Collect system, network, process, session data
        │
        ▼
Look for anomalies — things that don't belong
        │
        ▼
Narrow down to suspected machines
        │
        ▼
Deep dive evidence collection on those machines only
```

---

## Lab Environment

| Role | Machine | IP |
|------|---------|-----|
| Investigation Workstation | Client01 | 192.168.238.142 |
| Target | W02 | 192.168.238.144 |
| Target | W03 | 192.168.238.145 |

---

## Setup

```powershell
# Enable WinRM on investigation workstation
winrm quickconfig -q

# Add targets to TrustedHosts (required when not domain joined)
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "192.168.238.144,192.168.238.145" -Force

# Store credentials for reuse
$cred = Get-Credential
```

---

## Detection Commands

### 1 — System Information

```powershell
Invoke-Command -ComputerName 192.168.238.144,192.168.238.145 -Credential $cred -ScriptBlock {
    Get-ComputerInfo | Select-Object CsName, WindowsVersion, OsBuildNumber,
        CsDomain, CsPartOfDomain, OsLastBootUpTime, TimeZone, CsPhyicallyInstalledMemory
}
```

**What to look for:**
- OS version and build inconsistencies across machines
- Machines not joined to expected domain (CsPartOfDomain = False)
- Unusual last boot times — reboots outside maintenance windows
- Timezone differences that complicate timeline correlation

---

### 2 — Network Connections

```powershell
Invoke-Command -ComputerName 192.168.238.144,192.168.238.145 -Credential $cred -ScriptBlock {
    netstat -ano
}
```

**What to look for:**
- Unknown listening ports not present on other machines
- ESTABLISHED connections to external or unknown IPs
- Inconsistencies between machines with identical roles
- Port 4444, 1234, 8080 — common attacker/C2 ports

---

### 3 — Running Processes

```powershell
Invoke-Command -ComputerName 192.168.238.144,192.168.238.145 -Credential $cred -ScriptBlock {
    tasklist
}
```

**What to look for:**
- Unknown process names — especially ones mimicking system processes
- Processes running from Temp, AppData, or unusual directories
- Process present on one machine but not on others with identical roles
- High memory usage from unknown processes
- Classic attacker tools: mimikatz, meterpreter, nc, ncat

---

### 4 — Active Sessions

```powershell
Invoke-Command -ComputerName 192.168.238.144,192.168.238.145 -Credential $cred -ScriptBlock {
    query user
}
```

**What to look for:**
- Unknown usernames logged in
- Sessions active outside business hours
- Multiple concurrent sessions from same account
- Sessions from unexpected source machines

---

### 5 — Track User Logon Events Across Environment

```powershell
Invoke-Command -ComputerName 192.168.238.144,192.168.238.145 -Credential $cred -ScriptBlock {
    Get-WinEvent -FilterHashtable @{LogName="Security"; Id=4624} |
    Where-Object {
        $_.Properties[8].Value -eq 3 -or
        $_.Properties[8].Value -eq 10
    } |
    Select-Object TimeCreated,
        @{Name="User";Expression={$_.Properties[5].Value}},
        @{Name="LogonType";Expression={$_.Properties[8].Value}}
}
```

**Logon Types:**

| Type | Name | Meaning |
|------|------|---------|
| 2 | Interactive | Physical keyboard logon |
| 3 | Network | SMB, PowerShell Remoting |
| 4 | Batch | Scheduled task |
| 5 | Service | Service startup |
| 10 | RemoteInteractive | RDP session |

**What to look for:**
- Type 10 (RDP) from unexpected source IPs
- Type 3 logons from unknown machines
- Same account accessing multiple machines within seconds — lateral movement
- Logons from accounts that should not be active

---

## Lab 4 Findings — W02 and W03

### System Information Results

| Field | W02 | W03 | Assessment |
|-------|-----|-----|-----------|
| OS | Server 2019 Standard Eval | Server 2019 Standard Eval | ✅ Consistent |
| Build | 17763 | 17763 | ✅ Same build |
| Domain | WORKGROUP | WORKGROUP | ⚠️ Not domain joined |
| Last Boot | 7/20/2026 12:39 AM | 7/20/2026 12:40 AM | ⚠️ Midnight reboot |
| Timezone | UTC-08:00 Pacific | UTC-08:00 Pacific | ✅ Consistent |
| RAM | 1GB | 1GB | ✅ Expected |

### Network Connection Results

| Port | W02 | W03 | Notes |
|------|-----|-----|-------|
| 135 | ✅ | ✅ | RPC — normal Windows |
| 445 | ✅ | ✅ | SMB — normal Windows |
| 5985 | ✅ | ✅ | WinRM — expected, we enabled this |
| 139 | ✅ | ✅ | NetBIOS — normal |
| **3389** | **⚠️** | **❌** | **RDP only on W02 — inconsistency** |

### Running Process Results

Both machines show identical clean Windows baseline:
- System processes all verified legitimate
- MsMpEng.exe (Windows Defender) running on both
- wsmprovhost.exe visible — our PowerShell Remoting session
- No unknown executables detected
- No processes running from unusual paths

### Active Session Results

| Machine | User | Session | Logon Time | State |
|---------|------|---------|-----------|-------|
| W02 | Administrator | Console | 7/20/2026 12:40 AM | Active |
| W03 | Administrator | Console | 7/20/2026 12:41 AM | Active |

### Logon Event Results

| Machine | Time | User | Type | Assessment |
|---------|------|------|------|-----------|
| W02 | 12:40 AM | Administrator | 2 (Interactive) | Boot logon |
| W02 | 12:54 AM | Administrator | 3 (Network) | Our PS Remoting |
| W03 | 12:41 AM | Administrator | 2 (Interactive) | Boot logon |
| W03 | 12:54 AM | Administrator | 3 (Network) | Our PS Remoting |

---

## Anomalies Found

### Anomaly 1 — RDP Enabled on W02 Only

```
W02: Port 3389 LISTENING
W03: Port 3389 NOT PRESENT
```

Two servers with identical roles should have identical network profiles.
W02 has RDP exposed — larger attack surface.

**IR Action:** Check Security Event Log for Event ID 4657 (registry modification)
or System Log for when RDP registry key was changed. Determine if this was
admin action or attacker establishing persistent access.

```powershell
Invoke-Command -ComputerName 192.168.238.144 -Credential $cred -ScriptBlock {
    Get-WinEvent -FilterHashtable @{LogName='System'} |
    Where-Object { $_.Message -match 'Remote Desktop|TermService|fDenyTSConnections' } |
    Select-Object TimeCreated, Message | Select-Object -First 10
}
```

---

### Anomaly 2 — Simultaneous Midnight Reboot

```
W02 last boot: 7/20/2026 12:39 AM
W03 last boot: 7/20/2026 12:40 AM
Both within 1 minute of each other
Administrator logged in 1 minute after each reboot
```

Simultaneous reboots of multiple servers at midnight outside maintenance
windows is suspicious. Could indicate:
- Attacker pushed reboot command to clear volatile evidence
- Ransomware rebooting after installing encryption driver
- Legitimate maintenance — but should be verified

**IR Action:** Check Event ID 1074 (shutdown reason) on both machines.

```powershell
Invoke-Command -ComputerName 192.168.238.144,192.168.238.145 -Credential $cred -ScriptBlock {
    Get-WinEvent -FilterHashtable @{LogName='System'; Id=1074} |
    Select-Object TimeCreated, Message | Select-Object -First 5
}
```

---

## IR Correlation Timeline

```
Both machines rebooted at midnight (System Info — OsLastBootUpTime)
        │
        ▼
Administrator logged in 1 minute after each reboot (Active Sessions + Event 4624)
        │
        ▼
W02 has RDP port 3389 open — W03 does not (netstat -ano)
        │
        ▼
Logon events show only Administrator activity — no other accounts
        │
        ▼
QUESTIONS:
  Who rebooted these machines at midnight?
  Who enabled RDP on W02 and when?
  Was this scheduled maintenance or attacker activity?
        │
        ▼
NEXT STEP: Check Event Logs for shutdown reason and RDP configuration change
```

---

## Key Lesson

The detection commands do not tell you what happened.
They show you what is different from what you expect.

The analyst asks:
- Why does W02 have RDP open but W03 does not?
- Why did both machines reboot at the same time at midnight?
- Who was at the console 1 minute after midnight?

Those questions drive the investigation forward.
