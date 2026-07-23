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

## Setup
```bash
cd ~/ir-ansible
```

### ansible.cfg
```ini
[defaults]
host_key_checking = False
```

### hosts.ini
```ini
[linux_hosts]
192.168.238.160 ansible_user=ghostops ansible_password=Pro@123
192.168.238.162 ansible_user=ghostops ansible_password=Pro@123
```

### command.yml
```yaml
---
- name: Execute command on remote machines
  hosts: linux_hosts
  tasks:
  - name: Get Hostname
    command: hostname
    register: command_output
  - name: Display command output
    debug:
      var: command_output.stdout_lines
```

## Run
```bash
ansible-playbook -i hosts.ini command.yml
```

## Lab 5 Results — Environment Baseline

| Machine | IP | Hostname | Time | Timezone |
|---------|-----|---------|------|---------|
| WebServer | 192.168.238.160 | webserver | Thu Jul 23 07:27:14 UTC 2026 | Etc/UTC |
| FTPServer | 192.168.238.162 | ftpserver | Thu Jul 23 07:27:14 UTC 2026 | Etc/UTC |

## IR Notes
- Both machines UTC — timestamps directly comparable
- Time synchronized across both servers
- Ansible pulls information remotely via SSH — no console access needed
- Change command in command.yml to run any Linux command across all targets
