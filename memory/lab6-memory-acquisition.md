# Lab 6 — Remote Memory Acquisition

## Objective
Acquire RAM dump from W02 remotely and retrieve it to investigation workstation.

## Environment
| Role | Machine | IP |
|------|---------|-----|
| Investigation Workstation | Client01 | 192.168.238.142 |
| Target | W02 | 192.168.238.144 |

## Step 1 — Set Credentials
```powershell
$cred = Get-Credential
# Enter: Administrator / your password
```

## Step 2 — Create Output Directory on W02
```powershell
Invoke-Command -ComputerName 192.168.238.144 -Credential $cred -ScriptBlock {
    New-Item -ItemType Directory -Path "C:\IR\dmp" -Force
}
```

## Step 3 — Acquire RAM on W02
```powershell
Invoke-Command -ComputerName 192.168.238.144 -Credential $cred -ScriptBlock {
    $ss = Get-CimInstance -ClassName MSFT_StorageSubSystem -Namespace Root\Microsoft\Windows\Storage
    Invoke-CimMethod -InputObject $ss -MethodName "GetDiagnosticInfo" -Arguments @{
        DestinationPath="C:\IR\dmp"
        IncludeLiveDump=$true
    }
}
```

ReturnValue 0 = success.

## Step 4 — Verify Dump on W02
```powershell
Invoke-Command -ComputerName 192.168.238.144 -Credential $cred -ScriptBlock {
    ls C:\IR\dmp\localhost\
}
```

Result:
```
LiveDump.dmp        359,911,424 bytes
OperationalLog.evtx   1,118,208 bytes
```

## Step 5 — Create Local IR Directory
```powershell
New-Item -ItemType Directory -Path "C:\IR" -Force
```

## Step 6 — Retrieve Dump to Client01
```powershell
$session = New-PSSession -ComputerName 192.168.238.144 -Credential $cred
Copy-Item -FromSession $session -Path "C:\IR\dmp\localhost\LiveDump.dmp" -Destination "C:\IR\W02-memory.dmp"
```

## Step 7 — Verify Local Copy
```powershell
ls C:\IR\
# W02-memory.dmp  359,911,424 bytes
```

## Result
- RAM dump acquired remotely — no physical access to W02 required
- 359MB LiveDump.dmp retrieved to C:\IR\W02-memory.dmp
- Ready for analysis with MemProcFS or Volatility

## Next Steps — Analyse with MemProcFS
```powershell
# On IR_windows with MemProcFS installed
cd C:\Users\ir_user\Desktop\Tools\MemProcFS
.\MemProcFS.exe -f C:\IR\W02-memory.dmp -forensic 1
```

## Key Lesson
No third party tools needed on the target machine.
Native PowerShell CimInstance method works on any modern Windows Server.
RAM acquired and retrieved entirely over the network via PowerShell Remoting.
