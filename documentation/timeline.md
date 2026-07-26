# Incident Timeline — INC-XXXX

## How to Use This Template

Every event gets one row. Be precise with timestamps.
Use UTC or note the timezone consistently throughout.
Source = where the evidence came from (Sysmon, auth.log, netstat, etc.)

---

## Timeline

| UTC Timestamp | Event | Source | System | Analyst Notes |
|---------------|-------|--------|--------|--------------|
| | | | | |

---

## Attack Phase Markers

Use these labels in the Event column to mark attack phases:

```
[INITIAL ACCESS]      — attacker first gained entry
[EXECUTION]           — attacker ran code
[PERSISTENCE]         — attacker established persistence
[PRIVILEGE ESCALATION]— attacker gained higher privileges
[LATERAL MOVEMENT]    — attacker moved to another system
[COLLECTION]          — attacker accessed/staged data
[EXFILTRATION]        — attacker sent data out
[DETECTION]           — alert fired / analyst noticed
[CONTAINMENT]         — IR team took containment action
[ERADICATION]         — threat removed
[RECOVERY]            — systems restored
```

---

## Example Completed Timeline

| UTC Timestamp | Event | Source | System | Notes |
|---------------|-------|--------|--------|-------|
| 2026-07-20 00:39 | [INITIAL ACCESS] RDP connection from 185.x.x.x | Security Event 4624 | W02 | LogonType 10, unusual hour |
| 2026-07-20 00:40 | [EXECUTION] powershell.exe spawned by explorer.exe | Sysmon Event 1 | W02 | Encoded command line |
| 2026-07-20 00:41 | [PERSISTENCE] Scheduled task created | Sysmon Event 12 | W02 | Task name: WindowsUpdate |
| 2026-07-20 00:42 | [LATERAL MOVEMENT] SMB connection to W03 | Sysmon Event 3 | W02 | Port 445 |
| 2026-07-20 00:45 | [DETECTION] Alert fired in SIEM | Wazuh Rule 100014 | SIEMServer | Analyst notified |
| 2026-07-20 00:52 | [CONTAINMENT] W02 network isolated | Firewall rule | W02 | IR lead approved |
