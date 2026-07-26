# Traffic Analysis — tshark

## What tshark Is

tshark is the terminal version of Wireshark — the world's most widely used
network protocol analyser. During IR it captures network traffic to detect:

- Active C2 communication
- Lateral movement between machines
- Data exfiltration
- Attacker scanning activity
- Malware beaconing patterns

**Key principle:** Logs tell you what the OS recorded. Traffic tells you
what the machine actually did. A process can hide from logs but cannot
hide from a packet capture.

---

## Installation

```bash
sudo apt install tshark -y
# When asked: allow non-superusers to capture = Yes
```

---

## Two Types of Filters

tshark has two completely different filter systems that operate at different levels:

| Filter Type | Flag | Level | Syntax | Purpose |
|-------------|------|-------|--------|---------|
| Capture filter | `-f` | Packet acquisition | BPF syntax | Determines what gets captured — discards non-matching packets |
| Display filter | `-Y` | Packet display | Wireshark syntax | Filters already-captured packets for display |

**When to use which:**
- Use **capture filters** when you know exactly what you want — reduces data volume
- Use **display filters** when investigating a saved file — more expressive syntax
- In IR: save everything with no filter, then use display filters to analyse

---

## Exercise 1 — Capturing Traffic

### Capture from a specific interface
```bash
sudo tshark -i ens33
```

### Capture traffic from a specific host
```bash
sudo tshark -i ens33 host 8.8.8.8
```

### Save captured traffic to a file
```bash
sudo tshark -i ens33 -w /tmp/capture.pcap
```

### Save with time limit (10 seconds)
```bash
sudo tshark -i ens33 -w /tmp/capture.pcap -a duration:10
sudo chmod 644 /tmp/capture.pcap
```

### Read a saved capture file
```bash
tshark -r /tmp/capture.pcap
```

---

## Exercise 2 — Capture Filters (-f)

Capture filters use **Berkeley Packet Filter (BPF)** syntax.
They operate at the NIC level — non-matching packets are discarded entirely.

### Capture only TCP packets
```bash
sudo tshark -i ens33 -f "tcp"
```

### Capture packets on a specific port
```bash
sudo tshark -i ens33 -f "port 80"
sudo tshark -i ens33 -f "port 22"
sudo tshark -i ens33 -f "port 443"
```

### Capture packets with a source port
```bash
sudo tshark -i ens33 -f "src port 34934"
```

### Capture packets with a destination port
```bash
sudo tshark -i ens33 -f "dst port 443"
```

### Capture traffic between two specific IPs
```bash
sudo tshark -i ens33 -f "src host 192.168.238.132 and dst host 192.168.238.144"
```

### Capture only lab network traffic
```bash
sudo tshark -i ens33 -f "net 192.168.238.0/24"
```

### IR-specific capture filters
```bash
# Watch for connections to known C2 port
sudo tshark -i ens33 -f "dst port 4444"

# Watch for DNS queries only
sudo tshark -i ens33 -f "port 53"

# Watch for suspicious outbound HTTPS
sudo tshark -i ens33 -f "dst port 443 and not src net 192.168.238.0/24"
```

---

## Exercise 3 — Display Filters (-Y)

Display filters use **Wireshark syntax** — more expressive and protocol-aware.
They operate on already-captured packets — all packets are still saved.

### HTTP GET requests
```bash
sudo tshark -i ens33 -Y "http.request.method == GET"
```

### DNS requests only (not responses)
```bash
tshark -r /tmp/capture.pcap -Y "dns.flags.response == 0"
```

### All DNS traffic
```bash
tshark -r /tmp/capture.pcap -Y "dns"
```

### ICMP traffic (ping, traceroute)
```bash
sudo tshark -i ens33 -Y "icmp"
```

### DHCP traffic
```bash
sudo tshark -i ens33 -Y "bootp"
```

### HTTP to a specific host
```bash
sudo tshark -i ens33 -Y "http.host == 'www.example.com'"
```

### TCP SYN+ACK (connection responses — port scan detection)
```bash
tshark -r /tmp/capture.pcap -Y "tcp.flags.syn == 1 && tcp.flags.ack == 1"
```

### TCP SYN only (connection attempts — scanning)
```bash
tshark -r /tmp/capture.pcap -Y "tcp.flags.syn == 1 && tcp.flags.ack == 0"
```

### TLS version filtering
```bash
# TLSv1.2
tshark -r /tmp/capture.pcap -Y "ssl.record.version == 0x0303"

# TLSv1.3
tshark -r /tmp/capture.pcap -Y "tls.record.version == 0x0303"
```

### Filter by IP address
```bash
tshark -r /tmp/capture.pcap -Y "ip.addr == 192.168.238.144"
tshark -r /tmp/capture.pcap -Y "ip.src == 192.168.238.144"
tshark -r /tmp/capture.pcap -Y "ip.dst == 192.168.238.144"
```

### Filter by port
```bash
tshark -r /tmp/capture.pcap -Y "tcp.port == 22"
tshark -r /tmp/capture.pcap -Y "tcp.port == 4444"
```

---

## IR-Specific Display Filter Cheatsheet

| What You Are Hunting | Display Filter |
|---------------------|---------------|
| C2 on common port | `tcp.port == 4444 \|\| tcp.port == 1234 \|\| tcp.port == 8080` |
| DNS tunneling | `dns && frame.len > 200` |
| Port scanning | `tcp.flags.syn == 1 && tcp.flags.ack == 0` |
| SSH brute force | `tcp.port == 22 && tcp.flags.syn == 1` |
| Data exfiltration | `ip.dst != 192.168.238.0/24 && frame.len > 1400` |
| ICMP tunnel | `icmp && frame.len > 100` |
| Unencrypted credentials | `http \|\| ftp \|\| telnet` |
| Beaconing (regular intervals) | `ip.dst == suspicious_ip` |

---

## Practical IR Workflow

```bash
# Step 1 — Start capture immediately on compromised network segment
sudo tshark -i ens33 -w /tmp/ir-capture-$(date +%Y%m%d_%H%M%S).pcap

# Step 2 — Let it run while you investigate (open second terminal)

# Step 3 — Stop capture when done (Ctrl+C)

# Step 4 — Analyse saved file with targeted filters
sudo chmod 644 /tmp/ir-capture-*.pcap

# Find all unique destination IPs
tshark -r /tmp/ir-capture.pcap -T fields -e ip.dst | sort -u

# Find all DNS queries
tshark -r /tmp/ir-capture.pcap -Y "dns.flags.response == 0" -T fields -e dns.qry.name | sort -u

# Find connections to non-local IPs
tshark -r /tmp/ir-capture.pcap -Y "ip.dst != 192.168.238.0/24 && ip.dst != 127.0.0.0/8"
```

---

## Lab Results

**Interface:** ens33 (192.168.238.132)

**Exercise 1 — Live capture:**
- 8632 packets captured in 10 seconds
- Mix of SSH, HTTPS, ARP, DNS traffic

**Exercise 2 — SSH capture filter:**
- Isolated SSH traffic: 192.168.238.132 ↔ 192.168.238.1
- Confirmed MobaXterm session visible in capture

**Exercise 3 — Display filter on saved file:**
- SSH display filter showed encrypted session packets
- W02 filter (192.168.238.144) returned no results — machine was idle

---

## Key Lesson

**Capture filters** reduce what you collect — use when you know the target.
**Display filters** refine what you see — use when analysing saved captures.

In IR: always save a full unfiltered capture first.
You cannot go back and recapture traffic you discarded.
Analyse with display filters after — you have everything.
