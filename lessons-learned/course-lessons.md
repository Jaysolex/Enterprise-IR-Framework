# Lessons Learned — Enterprise IR Operations

## Overview

This framework was built through a complete Enterprise IR course covering
remote investigation, evidence acquisition, and anomaly detection across
Windows and Linux environments without centralized SOC tooling.

These are the operational lessons that matter.

---

## Lesson 1 — Tools Are Built on Fundamentals

Everything in this course — Kansa, Ansible, KAPE, CyLR — is built on
two fundamentals:

- **Windows:** PowerShell Remoting over WinRM (port 5985)
- **Linux:** SSH (port 22)

```
Kansa         = PowerShell scripts + WinRM
Ansible       = YAML playbooks + SSH
KAPE remote   = KAPE binary + WinRM copy + execution
CyLR remote   = CyLR binary + WinRM copy + execution
RAM dump      = Native PowerShell CimInstance + WinRM
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
Invoke-Command -ComputerName (get all 500) -ScriptBlock {netstat -ano}
```

One command. 500 results. 2 minutes.

Every technique in this framework solves the scale problem.
That is the entire point.

---

## Lesson 3 — Evidence Order Matters

Evidence volatility determines collection order.
Get this wrong and evidence is gone permanently.

```
1. RAM
   Why first: disappears completely on reboot
   Contains: running malware, decrypted credentials, C2 strings,
             encryption keys, process memory

2. Network state
   Why second: active connections drop as soon as attacker notices IR
   Contains: C2 connections, lateral movement in progress

3. Running processes
   Why third: malware may detect IR tools and self-terminate
   Contains: process names, parent-child relationships, command lines

4. Disk artifacts
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

## Lesson 6 — Anomalies Tell a Story

The tools give you data. The analyst finds the story.

Individual anomalies prove nothing:
```
W02 has RDP enabled          ← so what?
Both machines rebooted at midnight  ← maybe maintenance?
Administrator logged in 1 minute after reboot  ← normal?
No bash history on FTPServer ← freshly provisioned?
Port 6010 on FTPServer       ← X11 forwarding?
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
This is an active attacker covering tracks after establishing persistence
```

That correlation skill is what separates an IR analyst from someone
who reads event logs without knowing what they mean.

---

## Lesson 7 — Document Everything in Real Time

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

## Summary — The Seven Rules

```
1. Know the fundamentals (WinRM and SSH) — everything else builds on them
2. Solve for scale — one command, all machines, simultaneous
3. Collect in volatility order — RAM first, disk last
4. Use what is already there — built-in tools are enough
5. Detect before collecting — filter first, collect precisely
6. Correlate findings — individual anomalies mean little
7. Document everything in real time — documentation is the deliverable
```
