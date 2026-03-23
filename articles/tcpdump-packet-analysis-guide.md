---
layout: default
title: "How to Use tcpdump for Packet Analysis"
description: "Master tcpdump for capturing and analyzing network traffic: filter syntax, output formats, capture strategies, and practical security investigation exam..."
date: 2026-03-22
author: theluckystrike
permalink: /tcpdump-packet-analysis-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

How to Use tcpdump for Packet Analysis

tcpdump is the standard command-line packet analyzer on Linux and macOS. It captures raw network traffic and lets you filter, decode, and inspect it in real time or from saved capture files. This guide covers practical usage for security analysis. identifying what your devices are talking to, auditing unencrypted traffic, and investigating suspicious behavior.

Installation

tcpdump is usually pre-installed. If not:

```bash
Debian/Ubuntu
sudo apt install -y tcpdump

RHEL/CentOS/Fedora
sudo dnf install -y tcpdump

macOS (via Homebrew)
brew install tcpdump
```

Verify with `tcpdump --version`.

Basic Capture Commands

All captures require root or the `CAP_NET_RAW` capability.

```bash
Capture on a specific interface
sudo tcpdump -i eth0

List available interfaces
sudo tcpdump -D

Capture on all interfaces
sudo tcpdump -i any

Suppress DNS resolution (faster, shows raw IPs)
sudo tcpdump -n -i eth0

Suppress port name resolution too
sudo tcpdump -nn -i eth0

Show packet data in hex and ASCII
sudo tcpdump -XX -i eth0
```

Writing Captures to Files

Always save captures for offline analysis. PCAP files can be loaded into Wireshark:

```bash
Write to file
sudo tcpdump -i eth0 -w /tmp/capture.pcap

Read back a saved capture
tcpdump -r /tmp/capture.pcap

Rotate files every 100MB, keeping last 5
sudo tcpdump -i eth0 -w /tmp/cap.pcap -C 100 -W 5
```

Filter Syntax

BPF (Berkeley Packet Filter) expressions let you capture exactly what you need.

```bash
Filter by host
sudo tcpdump -nn -i eth0 host 192.168.1.50

Filter by destination only
sudo tcpdump -nn -i eth0 dst host 8.8.8.8

Filter by port
sudo tcpdump -nn -i eth0 port 443

Filter by protocol
sudo tcpdump -nn -i eth0 udp
sudo tcpdump -nn -i eth0 icmp

Capture DNS queries only
sudo tcpdump -nn -i eth0 port 53

Capture HTTP traffic (unencrypted)
sudo tcpdump -nn -i eth0 tcp port 80

Combine filters with and/or/not
sudo tcpdump -nn -i eth0 host 192.168.1.50 and port 443

Exclude SSH to avoid noise when SSHed into the machine
sudo tcpdump -nn -i eth0 not port 22
```

Practical Security Investigation Scenarios

1. Find All External Connections a Device Makes

Useful for auditing what a smart TV, IoT device, or new app phones home to:

```bash
Replace 192.168.1.55 with the target device's IP
sudo tcpdump -nn -i eth0 src host 192.168.1.55 \
  and not dst net 192.168.0.0/16 \
  -w /tmp/device-audit.pcap

While running, let the device sit idle for 10 minutes
Then read the capture and extract unique destinations
tcpdump -nn -r /tmp/device-audit.pcap | \
  awk '{print $5}' | \
  sed 's/\.[0-9]*$//' | \
  sort -u
```

2. Detect Unencrypted HTTP Traffic

```bash
Capture any HTTP traffic from your network
sudo tcpdump -nn -i eth0 tcp port 80 -A 2>/dev/null | \
  grep -E "^(GET|POST|PUT|DELETE|HEAD|Host:)"
```

Any output here represents cleartext HTTP. everything visible to your ISP, router, and anyone on the path.

3. Audit DNS Queries

DNS is typically unencrypted and reveals every domain your devices look up:

```bash
Capture all DNS queries
sudo tcpdump -nn -i eth0 port 53 -l | \
  grep -E " A | AAAA " | \
  awk '{print $NF}' | \
  sort -u | head -50
```

This shows domains being resolved in real time. Cross-reference against known telemetry endpoints to identify leaky apps.

4. Detect Port Scans

```bash
Capture TCP SYN packets (connection attempts) destined for your machine
sudo tcpdump -nn -i eth0 'tcp[tcpflags] & tcp-syn != 0' \
  and tcp[tcpflags] & tcp-ack == 0 \
  -c 100

If you see rapid connections to many ports from one source. that's a scan
```

5. Capture and Decode ICMP Traffic

```bash
Watch for ICMP packets. useful for detecting ping sweeps
sudo tcpdump -nn -i eth0 icmp -v
```

Extracting Content from Captures

For unencrypted traffic, you can extract payloads:

```bash
Print full packet content in ASCII
sudo tcpdump -nn -i eth0 -A port 80

Extract HTTP host headers
sudo tcpdump -nn -i eth0 -A port 80 2>/dev/null | \
  grep "^Host:" | sort -u

Save capture and analyze with strings
sudo tcpdump -i eth0 port 80 -w /tmp/http.pcap
strings /tmp/http.pcap | grep -E "^(GET|POST|HTTP|Host:|Cookie:)"
```

Verbose Output for Protocol Analysis

```bash
-v adds TTL, packet length, IP options
-vv adds more detail for DNS, NFS, etc.
-vvv shows maximum detail

sudo tcpdump -i eth0 -vv port 53 -n

Sample output:
14:22:31.412345 IP 192.168.1.5.41234 > 1.1.1.1.53: 12345+ A? example.com. (29)
14:22:31.423456 IP 1.1.1.1.53 > 192.168.1.5.41234: 12345 1/0/0 A 93.184.216.34 (45)
```

Automating Periodic Captures

Capture snapshots on a schedule for later review:

```bash
#!/bin/bash
/usr/local/bin/nightly-capture.sh
IFACE=eth0
OUTDIR=/var/captures
DURATION=300  # seconds

mkdir -p "$OUTDIR"
STAMP=$(date +%Y%m%d-%H%M%S)

timeout $DURATION tcpdump \
  -nn -i "$IFACE" \
  -w "${OUTDIR}/capture-${STAMP}.pcap" \
  not port 22 2>/dev/null

echo "Capture saved: ${OUTDIR}/capture-${STAMP}.pcap"
```

```bash
Run nightly at 2am
echo "0 2 * * * root /usr/local/bin/nightly-capture.sh" | \
  sudo tee /etc/cron.d/nightly-capture
```

Capture Storage Warning

PCAPs can grow quickly. At 100 Mbps traffic, a 5-minute capture can be several GB. Always use `-C` for size rotation and `-W` for file count limits, or run captures for limited durations.

Related Reading

- [How to Set Up ntopng for Network Analytics](/ntopng-network-analytics-setup-guide/)
- [How to Use strace for Security Analysis](/strace-security-analysis-guide/)
- [VPN Kill Switch with iptables on Linux](/vpn-kill-switch-linux-iptables-setup/)

---

Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
