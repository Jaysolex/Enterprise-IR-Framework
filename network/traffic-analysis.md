# Traffic Analysis — tshark

## What tshark Is
tshark is the terminal version of Wireshark. During IR it captures
network traffic to detect C2 communication, lateral movement,
data exfiltration, and attacker scanning activity.

## Installation
```bash
sudo apt install tshark -y
```

## Lab 7 — Three Exercises

### Exercise 1 — Capture Live Traffic
```bash
sudo tshark -i ens33
```

### Exercise 2 — Capture Filters (filter DURING capture)
```bash
# Only SSH traffic
sudo tshark -i ens33 -f "port 22"

# Only lab network traffic
sudo tshark -i ens33 -f "net 192.168.238.0/24"

# Only traffic from specific host
sudo tshark -i ens33 -f "host 192.168.238.144"
```

### Exercise 3 — Display Filters (filter AFTER capture)
```bash
# Save capture to file
sudo tshark -i ens33 -w /tmp/capture.pcap -a duration:10
sudo chmod 644 /tmp/capture.pcap

# Filter saved capture
tshark -r /tmp/capture.pcap -Y "tcp.port == 22"
tshark -r /tmp/capture.pcap -Y "ip.addr == 192.168.238.144"
tshark -r /tmp/capture.pcap -Y "http"
tshark -r /tmp/capture.pcap -Y "dns"
```

## Capture vs Display Filters

| Type | When Applied | Purpose |
|------|-------------|---------|
| Capture filter (-f) | During capture | Reduce data collected |
| Display filter (-Y) | After capture | Analyse saved file |

## IR Use Cases

| Filter | What You Are Looking For |
|--------|------------------------|
| tcp.port == 4444 | Metasploit default C2 port |
| ip.addr == suspicious_ip | Traffic to known bad IP |
| dns | DNS queries — C2 domain resolution |
| http.request | Unencrypted web requests |
| tcp.flags.syn == 1 | Port scanning activity |

## Lab Results
- Interface: ens33
- SSH traffic captured: 192.168.238.132 ↔ 192.168.238.1
- Protocol: SSH encrypted (TLSv1.3 equivalent)
- W02 traffic: None during capture window (machine idle)

## Key Lesson
tshark sees all traffic on the interface.
Capture filters reduce noise during collection.
Display filters allow targeted analysis of saved captures.
In IR — save first with no filter, analyse later with display filters.
