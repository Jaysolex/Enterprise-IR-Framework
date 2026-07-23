# Linux Anomaly Detection

## Overview
Remote anomaly detection across Linux endpoints using Ansible over SSH.
No agents required — uses SSH and built-in Linux commands only.

## Lab Environment
| Role | Machine | IP |
|------|---------|-----|
| Investigation Workstation | SIEMServer | 192.168.238.132 |
| Target 1 | WebServer | 192.168.238.160 |
| Target 2 | FTPServer | 192.168.238.162 |

## Detection Commands

### A — Network Interfaces
```bash
command: ip addr
```
What to look for: unexpected interfaces, unusual IP ranges, unknown network adapters

### B — Active Connections
```bash
command: ss -tuln
```
What to look for: unknown listening ports, established connections to external IPs, C2 traffic

### C — Running Processes
```bash
command: ps aux
```
What to look for: unknown process names, processes from /tmp, crypto miners, reverse shells

### D — Login History
```bash
command: journalctl _SYSTEMD_UNIT=ssh.service --no-pager -n 20
```
What to look for: logins from unknown IPs, logins at unusual hours, unknown usernames

### E — Command History
```bash
command: cat /home/ghostops/.bash_history
```
What to look for: wget/curl downloading files, base64 commands, privilege escalation

## Lab 5 Findings

### Network Connections
| Port | WebServer | FTPServer | Notes |
|------|-----------|-----------|-------|
| 22 | ✅ | ✅ | SSH — expected |
| 53 | ✅ | ✅ | DNS local — normal |
| 323 | ✅ | ✅ | NTP chrony — normal |
| 6010 | ❌ | ⚠️ | X11 forwarding on FTPServer |

### Running Processes
Both machines show identical clean process baseline.
No unknown executables detected on either machine.

### Login History
| Machine | Login Sources | Assessment |
|---------|-------------|-----------|
| WebServer | 192.168.238.132, 192.168.238.1 | ✅ Known sources only |
| FTPServer | 192.168.238.132, 192.168.238.1 | ✅ Known sources only |

### Command History
| Machine | Finding | Assessment |
|---------|---------|-----------|
| WebServer | ip, whoami, apt install | ✅ Clean — normal admin commands |
| FTPServer | No history file | ⚠️ Investigate — cleared or fresh |

## Anomalies Found

### Anomaly 1 — Port 6010 on FTPServer
X11 forwarding port open on FTPServer.
Result of SSH session with X11 forwarding enabled.
In a real IR — verify who opened this and why.

### Anomaly 2 — No Bash History on FTPServer
No .bash_history file exists on FTPServer.
Could indicate: fresh machine, history cleared by attacker, or no interactive logins.
Action: Check journalctl for all commands run on this machine.

## IR Correlation
FTPServer has no bash history (Command History)
+
FTPServer has port 6010 open (Network)
|
v
Questions: Was someone on this machine recently?
Was history deliberately cleared?
Who connected via X11 forwarding and why?
|
v
Next step: Check journalctl for full activity log
## Key Lesson
One command on SIEMServer pulls investigation data from
all Linux targets simultaneously via SSH.
No agents, no special tools — just Ansible and SSH.
