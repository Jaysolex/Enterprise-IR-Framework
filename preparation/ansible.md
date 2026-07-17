# Ansible — Remote Linux Investigation

## What Ansible Is
Ansible uses SSH to execute commands across multiple Linux machines
simultaneously from a single investigation workstation.
Linux equivalent of Kansa for Windows.

## Lab Environment
- Investigation machine: IR_linux (10.1.60.105)
- Target 1: Web server (10.1.200.140)
- Target 2: FTP server (10.1.255.195)

## Installation
```bash
sudo apt install ansible -y
```

## Three Required Files

### 1 — ansible.cfg
```ini
[defaults]
host_key_checking = False
```

### 2 — hosts.ini
```ini
[linux_hosts]
10.1.200.140 ansible_user=ubuntu ansible_password=Pro@123
10.1.255.195 ansible_user=ubuntu ansible_password=Pro@123
```

### 3 — command.yml
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

## Lab 3 Results — Environment Baseline

| Machine | IP | Hostname | Time | Timezone |
|---------|-----|---------|------|---------|
| Web | 10.1.200.140 | ip-10-1-200-140 | Fri Jul 17 14:42:41 UTC 2026 | Etc/UTC |
| FTP | 10.1.255.195 | ip-10-1-255-195 | Fri Jul 17 14:42:41 UTC 2026 | Etc/UTC |

## IR Notes
- All machines UTC — timestamps directly comparable, no conversion needed
- Time synchronized — consistent evidence timeline
- Establish baseline at start of every Linux IR investigation
- Change command in command.yml to run any Linux command across all targets
