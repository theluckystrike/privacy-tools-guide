---
layout: default
title: "How to Set Up Suricata in IPS Mode"
description: "Configure Suricata as an inline IPS using NFQUEUE on Linux to actively drop malicious traffic with ET Pro rules, custom signatures, and fail-open handling"
date: 2026-03-22
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /suricata-ips-mode-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
How to Set Up Suricata in IPS Mode

Suricata in IDS mode logs alerts but lets traffic through. IPS mode uses the Linux kernel's NFQUEUE to intercept packets before delivery, letting Suricata drop matching traffic in real time. This guide covers the NFQUEUE setup, inline rule tuning, and fail-open handling so a Suricata crash never takes down your network.

IDS vs IPS Mode

| Mode | Mechanism | Traffic Impact |
|---|---|---|
| IDS (pcap/af-packet) | Passive copy | None. read-only |
| IPS (NFQUEUE) | Kernel intercepts; Suricata decides verdict | Blocks matching traffic |

IPS mode adds latency (0.5, 2 ms typical) and a new failure mode: if Suricata crashes, traffic stops unless you configure fail-open.

---

1. Install Suricata

```bash
sudo add-apt-repository ppa:oisf/suricata-stable
sudo apt update && sudo apt install -y suricata suricata-update

suricata --version
```

---

2. Configure NFQUEUE

NFQUEUE is a netfilter target that holds packets in a kernel queue until an userspace process accepts or drops them. Add rules to send traffic through Suricata:

```bash
IPS on a forwarding host (router/firewall)
sudo iptables -I FORWARD -j NFQUEUE --queue-num 0 --queue-bypass

IPS on the host itself (protect this server)
sudo iptables -I INPUT  -j NFQUEUE --queue-num 0 --queue-bypass
sudo iptables -I OUTPUT -j NFQUEUE --queue-num 0 --queue-bypass

IPv6
sudo ip6tables -I INPUT  -j NFQUEUE --queue-num 0 --queue-bypass
sudo ip6tables -I OUTPUT -j NFQUEUE --queue-num 0 --queue-bypass

--queue-bypass is critical for fail-open:
if Suricata is not running, the kernel passes traffic without inspection
REMOVE --queue-bypass if you want fail-CLOSED (drop all traffic when Suricata is down)
```

Make iptables rules persistent:

```bash
sudo apt install -y iptables-persistent
sudo iptables-save > /etc/iptables/rules.v4
sudo ip6tables-save > /etc/iptables/rules.v6
```

---

3. Configure Suricata for IPS Mode

```yaml
/etc/suricata/suricata.yaml. relevant sections only

IPS via NFQUEUE
nfq:
  mode: accept          # accept packets by default; rules drop specific ones
  batchcount: 20        # packets to batch per syscall (performance tuning)
  fail-open: true       # if Suricata queue is full, let packets through

Disable IDS interfaces (we're using NFQUEUE, not a NIC)
af-packet:
  - interface: default
    # comment out or set enabled: false

Set default action for rules to 'drop' instead of 'alert' for IPS
(optional. controlled per-rule with 'drop' keyword)
action-order:
  - drop
  - pass
  - alert

Thread count for NFQUEUE workers
threading:
  set-cpu-affinity: yes
  cpu-affinity:
    - management-cpu-set:
        cpu: [0]
    - receive-cpu-set:
        cpu: [0]
    - worker-cpu-set:
        cpu: [1, 2, 3]
```

---

4. Download and Update Rules

```bash
suricata-update manages rulesets
sudo suricata-update

Add Emerging Threats Open (free, high quality)
sudo suricata-update add-source et/open \
  --url https://rules.emergingthreats.net/open/suricata-6.0/emerging.rules.tar.gz

Add ProofPoint ET Pro (paid, latest signatures)
sudo suricata-update add-source etpro/etpro --secret-code YOUR_CODE

Update all sources
sudo suricata-update --no-reload

Reload rules without restarting Suricata
sudo kill -USR2 $(cat /var/run/suricata.pid)
```

---

5. Start Suricata in IPS Mode

```bash
IPS mode - -q 0 = NFQUEUE 0
sudo suricata -c /etc/suricata/suricata.yaml -q 0 --pidfile /var/run/suricata.pid -D

sudo tail -f /var/log/suricata/suricata.log
Look for - "All threads are running"
```

Systemd service for IPS mode:

```ini
/etc/systemd/system/suricata-ips.service
[Unit]
Description=Suricata IPS (NFQUEUE mode)
After=network.target

[Service]
Type=simple
ExecStartPre=/sbin/iptables -I INPUT -j NFQUEUE --queue-num 0 --queue-bypass
ExecStartPre=/sbin/iptables -I OUTPUT -j NFQUEUE --queue-num 0 --queue-bypass
ExecStart=/usr/bin/suricata -c /etc/suricata/suricata.yaml -q 0
ExecStopPost=/sbin/iptables -D INPUT -j NFQUEUE --queue-num 0 --queue-bypass
ExecStopPost=/sbin/iptables -D OUTPUT -j NFQUEUE --queue-num 0 --queue-bypass
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now suricata-ips
```

---

6. Verify Drops Are Working

```bash
Check stats. blocked packet count
sudo suricatasc -c "dump-counters" | python3 -m json.tool | grep -i drop

Or via stats log
sudo tail /var/log/suricata/stats.log | grep drop

View fast alerts (drops show as [Drop])
sudo tail -f /var/log/suricata/fast.log
```

Test with a known-bad signature:

```bash
EICAR test string. triggers AV/IPS rules
curl -s http://www.eicar.org/download/eicar.com 2>&1 || echo "Blocked"
If IPS is working - connection refused or empty response
```

---

7. Write IPS Drop Rules

Use `drop` instead of `alert` to actively block traffic:

```bash
sudo tee /etc/suricata/rules/local.rules > /dev/null <<'EOF'
Drop Telnet connections
drop tcp any any -> $HOME_NET 23 \
  (msg:"IPS: DROP Telnet"; flow:to_server; sid:9100001; rev:1;)

Drop known C2 domain (replace with actual IOC)
drop dns any any -> any any \
  (msg:"IPS: DROP known malware C2"; dns.query; \
   content:"badactor.example.com"; nocase; \
   sid:9100002; rev:1;)

Drop large ICMP packets (ping-of-death signature)
drop icmp any any -> $HOME_NET any \
  (msg:"IPS: DROP large ICMP"; dsize:>1000; \
   sid:9100003; rev:1;)

Rate-limit SSH - drop if >10 SYNs in 30s from same source
drop tcp any any -> $HOME_NET 22 \
  (msg:"IPS: SSH brute force drop"; flags:S; \
   threshold:type threshold, track by_src, count 10, seconds 30; \
   sid:9100004; rev:1;)
EOF
```

Add to `suricata.yaml` under `rule-files`:

```yaml
rule-files:
  - suricata.rules
  - /etc/suricata/rules/local.rules
```

---

8. Tune False Positives: Convert drop to alert

When a drop rule fires on legitimate traffic, suppress it or convert to alert while you investigate:

```bash
Suppress a specific rule for an internal host
sudo tee /etc/suricata/threshold.conf > /dev/null <<'EOF'
Suppress rule 2001219 for the monitoring server
suppress gen_id 1, sig_id 2001219, track by_src, ip 10.0.0.50

Rate-limit an alert to once per hour per source
threshold gen_id 1, sig_id 2012648, type limit, track by_src, count 1, seconds 3600
EOF
```

In `suricata.yaml`:

```yaml
threshold-file: /etc/suricata/threshold.conf
```

---

9. Monitor Alerts in EVE JSON

Suricata's EVE JSON log is the richest output format:

```bash
View live drops
sudo tail -f /var/log/suricata/eve.json \
  | python3 -c "
import sys, json
for line in sys.stdin:
    e = json.loads(line)
    if e.get('event_type') == 'alert' and e.get('alert', {}).get('action') == 'blocked':
        a = e['alert']
        print(f\"{e['timestamp']} DROP [{a['signature_id']}] {a['signature']} \
{e.get('src_ip')}:{e.get('src_port')} -> {e.get('dest_ip')}:{e.get('dest_port')}\")
"
```

---

Multi-Queue Performance Scaling

For high-throughput (>1 Gbps) networks:

```bash
Use multiple NFQUEUE queues (0, 3) with load balancing
sudo iptables -I FORWARD -j NFQUEUE --queue-balance 0:3 --queue-bypass

Start Suricata with 4 workers
sudo suricata -c /etc/suricata/suricata.yaml \
  -q 0 -q 1 -q 2 -q 3 -D
```

---

Related Reading

- [How to Set Up Snort IDS on Linux](/snort-ids-linux-setup-guide/)
- [How to Detect ARP Spoofing Attacks](/detect-arp-spoofing-attacks-guide/)
- [How to Set Up CrowdSec for Server Security](/crowdsec-server-security-setup-guide/)

---

Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
