# soc-day02-wireshark-syn-scan-detection
Wireshark live traffic capture — SYN scan detection, packet analysis, and half-open scan behavior documented using Kali Linux
# SOC Day 02 — Wireshark SYN Scan Detection

Wireshark live traffic capture — SYN scan detection, packet analysis, 
and half-open scan behavior documented using Kali Linux

## Lab Overview

| Field | Details |
|---|---|
| Attack Type | TCP SYN Scan — Half-Open Scan |
| Detection Platform | Wireshark 4.6.3 |
| Capture Interface | eth0 |
| Target | scanme.nmap.org (45.33.32.156) |
| Scan Source | Kali Linux running Nmap 7.98 with -sS flag |
| Total Packets Captured | 2120 |
| SYN Packets Filtered | 2002 (94.4% of total traffic) |
| Outcome | SYN scan successfully detected — half-open behavior confirmed via RST packet after SYN-ACK response |

## What Happened

A TCP SYN scan was run against scanme.nmap.org while Wireshark 
captured live traffic on eth0. The scan generated 2120 packets. 
After applying a display filter for SYN-only packets, 2002 packets 
were isolated — all originating from the same source IP hitting 
different destination ports in rapid succession with no completed 
handshakes.

## Wireshark Filter Used

tcp.flags.syn == 1 && tcp.flags.ack == 0

This filter isolates TCP segments with only the SYN flag set and 
no ACK — the exact signature of a half-open scan probe. It removes 
SYN-ACK responses and RST packets, showing only the outbound scan 
probes.

## Key Findings

### Finding 1 — SYN Flood Pattern Visible
One source IP (192.168.64.2) sent SYN packets to hundreds of 
different destination ports on 45.33.32.156 in rapid succession. 
Every packet was 58 bytes — tiny, no data payload, just the SYN 
flag. This volume and timing pattern is the signature of automated 
port scanning, not normal user traffic.

### Finding 2 — Half-Open Scan Behavior Confirmed
When port 22 responded with a SYN-ACK (confirming it was open), 
Nmap immediately sent an RST packet to kill the connection instead 
of completing the handshake with a final ACK. This confirms the 
half-open scan mechanism — the scanner deliberately chose not to 
complete the connection because it already had the information it 
needed (port is open) and completing the handshake would create a 
full session log.

### Finding 3 — Open Ports Identified
Cross-referencing Wireshark capture with Nmap output confirmed 
4 open ports:
- Port 22 — SSH
- Port 80 — HTTP  
- Port 9929 — nping-echo
- Port 31337 — Elite (known RAT history, high-interest port)

## MITRE ATT&CK Mapping

| Technique | ID | Description |
|---|---|---|
| Active Scanning: Port Scanning | T1595.001 | Nmap SYN scan used to enumerate open ports |
| Network Service Discovery | T1046 | Open services identified via scan results |

## Triage Note

**Observed:** Live Wireshark capture on eth0 detected 2002 SYN 
packets from 192.168.64.2 to 45.33.32.156 in under 10 seconds. 
RST packet observed immediately after SYN-ACK from port 22 — 
confirms half-open scan. No completed TCP handshakes observed.

**Severity:** Medium — target is authorized public test server. 
Same pattern on an internal host or unknown external IP would be 
Critical.

**Ruled out:** Confirmed authorized scan against scanme.nmap.org. 
Not a production asset.

**Recommendation:** In a real environment — block the source IP 
at the firewall, alert on the pattern in SIEM using threshold 
rule (50+ SYNs from one source in under 60 seconds with no 
completed handshakes), and escalate port 31337 activity 
immediately.

## Tools Used
- Wireshark 4.6.3
- Nmap 7.98
- Kali Linux terminal
- Display filter: tcp.flags.syn == 1 && tcp.flags.ack == 0
