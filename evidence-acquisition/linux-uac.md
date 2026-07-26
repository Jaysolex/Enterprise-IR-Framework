# Linux Evidence Acquisition — UAC

## What UAC Is
UAC (Unix-like Artifacts Collector) is the Linux equivalent of KAPE/CyLR.
It collects forensic artifacts from Linux systems into a single compressed archive.

## Installation
```bash
wget https://github.com/tclahr/uac/archive/refs/heads/main.zip
unzip main.zip
cd uac-main
```

## Available Profiles
```bash
./uac --profile list
```

| Profile | Purpose |
|---------|---------|
| full | Complete artifact collection |
| ir_triage | IR-focused collection (recommended) |
| offline | For mounted disk images |
| offline_ir_triage | Offline IR collection |

## Run IR Triage Collection
```bash
mkdir -p ~/evidence
sudo ./uac -p ir_triage ~/evidence/
```

## Lab Results — SIEMServer Collection
- Artifacts collected: 219
- Collection time: 1762 seconds (29 minutes)
- Output size: 911MB
- Output file: uac-SIEMServer-linux-20260725044438.tar.gz
- Hash log: uac-SIEMServer-linux-20260725044438.log

## What UAC Collects (219 Artifacts)
| Category | Examples |
|----------|---------|
| Process | ps, lsof, pstree, running process hashes |
| Network | arp, netstat, ss, iptables, ufw, ip |
| Users | passwd, shadow, groups, sudo, SSH keys |
| Persistence | cron, systemd, rc files, startup scripts |
| Logs | auth.log, syslog, kern.log, journald |
| Files | bash history, shell configs, SSH known_hosts |
| System | boot, /etc, /tmp, /dev/shm, utmp |

## Output Structure
uac-hostname-os-timestamp.tar.gz
├── live_response/
│ ├── process/
│ ├── network/
│ └── hardware/
├── files/
│ ├── logs/
│ ├── shell/
│ ├── ssh/
│ └── system/
└── bodyfile/
uac-hostname-os-timestamp.log ← acquisition log with hashes
## Deploy via Ansible Across Multiple Linux Targets
```yaml
---
- name: UAC Evidence Collection
  hosts: linux_hosts
  tasks:
  - name: Copy UAC to target
    copy:
      src: /home/ghostop/uac-main/
      dest: /tmp/uac/
      mode: '0755'
  - name: Run UAC triage
    command: /tmp/uac/uac -p ir_triage /tmp/evidence/
    become: yes
```

## Key Advantage
One command collects 219 artifact categories automatically.
Output includes acquisition log with file hashes for chain of custody.
