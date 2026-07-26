# Lessons Learned — Enterprise IR Operations

## Overview

This framework was built through a complete Enterprise IR course covering
remote investigation, evidence acquisition, and anomaly detection across
Windows and Linux environments without centralized SOC tooling.

These are the operational lessons that matter.

---

## Lesson 1 — Tools Are Built on Fundamentals

Everything in this course is built on two protocols:

- **Windows:** PowerShell Remoting over WinRM (port 5985)
- **Linux:** SSH (port 22)

```
Kansa         = PowerShell scripts + WinRM
Ansible       = YAML playbooks + SSH
KAPE remote   = KAPE binary + WinRM copy + execution
CyLR remote   = CyLR binary + WinRM copy + execution
RAM dump      = Native PowerShell CimInstance + WinRM
UAC remote    = UAC binary + SSH + Ansible
LiME          = Kernel module compiled per target kernel
tshark        = Network capture on investigation machine
```

If you understand WinRM and SSH you can build any IR capability yourself.
The tools save time — the protocol knowledge makes you dangerous.

---

## Lesson 2 — Scale Is the Real Challenge

A single compromised machine is straightforward to investigate.
500 machines during an active ransomware incident is a different problem entirely.

Without scale:
```
RDP into W01 → investigate → log out (20 minutes)
RDP into W02 → investigate → log out (20 minutes)
...repeat 498 more times = 166 hours
```

The attacker is gone in 4 hours. You finish in 7 days.

With scale:
```
Invoke-Command -ComputerName (all 500) -ScriptBlock {netstat -ano}
ansible-playbook -i hosts.ini command.yml
```

One command. All machines. Simultaneous results.
Every technique in this framework solves the scale problem.

---

## Lesson 3 — Evidence Order Matters

Evidence volatility determines collection order.
Get this wrong and evidence is gone permanently.

```
1. RAM
   Tool: Windows = CimInstance native | Linux = LiME kernel module
   Why first: disappears completely on reboot
   Contains: running malware, decrypted credentials, C2 strings,
             encryption keys, process memory

2. Network state
   Tool: netstat -ano (Windows) | ss -tuln (Linux)
   Why second: active connections drop as soon as attacker notices IR
   Contains: C2 connections, lateral movement in progress

3. Running processes
   Tool: tasklist (Windows) | ps aux (Linux)
   Why third: malware may detect IR tools and self-terminate
   Contains: process names, parent-child relationships, command lines

4. Disk artifacts
   Tool: KAPE / CyLR (Windows) | UAC (Linux)
   Why last: persists across reboots, survives investigation
   Contains: event logs, registry, prefetch, LNK files, browser history
```

**Never reboot before acquiring RAM.**
That single rule preserves more evidence than any tool.

---

## Lesson 4 — No Fancy Tools Required

The entire framework uses:

**Windows built-ins:**
- PowerShell, WinRM, netstat -ano, tasklist, query user
- Get-ComputerInfo, ipconfig, Get-WinEvent

**Linux built-ins:**
- SSH, ps aux, ss -tuln, journalctl, bash_history, ip addr

**Free open source:**
- Kansa — PowerShell IR framework
- Ansible — SSH automation
- CyLR — zero-config evidence collection
- KAPE — modular artifact collection
- MemProcFS — memory forensics filesystem
- UAC — Linux artifact collection (219 categories)
- LiME — Linux kernel module RAM acquisition
- tshark — network traffic capture and analysis
- Sysmon — enhanced Windows event logging

Total cost: $0

You do not need a $100,000 SIEM to do enterprise IR.
You need to understand the OS.

---

## Lesson 5 — Detect First, Collect Second

A common mistake: collect everything from everywhere.

That approach:
- Takes 8+ hours for large environments
- Generates terabytes of data to analyse
- Misses the attacker while you are still collecting

The correct approach:
```
Run lightweight detection across ALL machines (5 minutes)
        │
        v
Get-ComputerInfo, netstat, tasklist, query user, ps aux, ss
        │
        v
Identify 3 suspicious machines out of 500
        │
        v
Run KAPE/CyLR/RAM dump on THOSE 3 machines only (30 minutes)
        │
        v
Analyse targeted evidence (focused, fast)
```

Detection is a filter. Collection is a cost.
Filter first. Collect precisely.

---

## Lesson 6 — Build Visibility Before Incidents Happen

Default Windows logging is insufficient for IR.
Sysmon transforms what you can see:

```
Default Event 4688:
  Process: powershell.exe
  That is all.

Sysmon Event 1:
  Process:     powershell.exe
  CommandLine: powershell.exe -enc JABjAG0AZAA...
  Hash:        SHA256=4A5B6C7D...
  Parent:      winword.exe
  ParentCmd:   WINWORD.EXE /n "malicious.docx"
  User:        DOMAIN\victim
```

Deploy Sysmon before incidents happen — not during.
SwiftOnSecurity config is the industry standard starting point.
Remote deployment via PowerShell Remoting takes 5 minutes across all machines.

---

## Lesson 7 — Traffic Analysis Reveals What Logs Cannot

Logs tell you what the OS recorded.
Network traffic tells you what the machine actually did.

```
Log says:     powershell.exe ran
Traffic shows: powershell.exe connected to 185.234.x.x:4444
```

tshark saves to pcap format — the universal standard readable by Wireshark.

```bash
# Save during IR — no filter, capture everything
sudo tshark -i ens33 -w /tmp/capture.pcap

# Analyse after — targeted display filters
tshark -r /tmp/capture.pcap -Y "ip.addr == suspicious_ip"
tshark -r /tmp/capture.pcap -Y "tcp.port == 4444"
tshark -r /tmp/capture.pcap -Y "dns"
```

Save first with no filter. Analyse later with display filters.
You cannot go back and recapture traffic you missed.

---

## Lesson 8 — Windows and Linux IR Are Parallel Disciplines

| Task | Windows | Linux |
|------|---------|-------|
| Remote investigation | PowerShell Remoting | Ansible + SSH |
| Evidence collection | KAPE / CyLR | UAC |
| RAM acquisition | CimInstance (native) | LiME (kernel module) |
| Memory analysis | MemProcFS + Volatility | Volatility |
| Persistence hunting | Registry, Tasks, Services | Cron, Systemd, RC files |
| Log analysis | Get-WinEvent | journalctl, auth.log |
| Visibility | Sysmon | auditd |
| Network capture | tshark | tshark |

Same investigative questions. Different tools. Same mindset.

---

## Lesson 9 — Anomalies Tell a Story

The tools give you data. The analyst finds the story.

Individual anomalies prove nothing:
```
W02 has RDP enabled                    ← so what?
Both machines rebooted at midnight     ← maybe maintenance?
Administrator logged in 1 min after   ← normal?
No bash history on FTPServer           ← freshly provisioned?
Port 6010 on FTPServer                 ← X11 forwarding?
```

Correlated anomalies tell the truth:
```
Both machines rebooted simultaneously at midnight (not a maintenance window)
        +
Administrator logged in within 1 minute on both (someone was ready and waiting)
        +
W02 has RDP enabled — W03 does not (inconsistent config — someone changed it)
        +
No bash history on FTPServer (someone cleared it)
        =
Active attacker covering tracks after establishing persistence
```

That correlation skill is what separates an IR analyst from someone
who reads event logs without knowing what they mean.

---

## Lesson 10 — Document Everything in Real Time

Documentation written after an incident from memory is unreliable.
Documentation written during an incident is evidence.

Every command you run: document it.
Every finding you make: document it.
Every timestamp you observe: document it.
Every decision you make: document why.

Your documentation becomes:
- The incident timeline
- The evidence chain of custody
- The executive report
- The lessons learned
- The improved playbook for next time

The analyst who cannot document findings clearly is only half useful.
The analyst who documents everything is irreplaceable.

---

## The Ten Rules

```
1.  Know the fundamentals — WinRM and SSH power everything
2.  Solve for scale — one command, all machines, simultaneous
3.  Collect in volatility order — RAM first, disk last
4.  Use what is already there — built-in tools are enough
5.  Detect before collecting — filter first, collect precisely
6.  Build visibility before incidents — deploy Sysmon now
7.  Capture traffic — logs miss what the network reveals
8.  Windows and Linux are parallel — same questions, different tools
9.  Correlate findings — individual anomalies mean little
10. Document everything in real time — documentation is the deliverable
```

---

## Tools Reference

| Tool | Platform | Purpose |
|------|---------|---------|
| PowerShell Remoting | Windows | Remote investigation and execution |
| Kansa | Windows | Modular IR evidence collection |
| KAPE | Windows | Forensic artifact collection |
| CyLR | Windows | Zero-config evidence collection |
| CimInstance | Windows | Native RAM acquisition |
| MemProcFS | Windows | Memory forensics filesystem |
| Sysmon | Windows | Enhanced event logging |
| Ansible | Linux | Remote investigation via SSH |
| UAC | Linux | Forensic artifact collection (219 categories) |
| LiME | Linux | Kernel module RAM acquisition |
| tshark | Both | Network traffic capture and analysis |
| Volatility | Both | Memory forensics analysis |
