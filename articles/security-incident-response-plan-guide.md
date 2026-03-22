---
layout: default
title: "How to Create a Security Incident Response Plan"
description: "Build a security incident response plan with detection criteria, escalation paths, containment procedures, and post-incident review templates for."
date: 2026-03-22
author: theluckystrike
permalink: /security-incident-response-plan-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Create a Security Incident Response Plan

A security incident response plan (IRP) defines exactly what to do when something goes wrong — before it happens. Without one, your team improvises under pressure, wastes time, and makes decisions that destroy forensic evidence. This guide covers the six phases of incident response with templates you can adapt.

## Key Takeaways

- **Output**: $OUTDIR"
tar czf "/tmp/triage-${TIMESTAMP}.tar.gz" "$OUTDIR"
```

## Phase 3: Containment

Containment stops the attack from spreading.
- **Topics covered**: the six phases, phase 1: preparation, incident classification
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements

## The Six Phases

```
1. Preparation     → Plan, tooling, contact list, practice
2. Identification  → Detect and confirm the incident
3. Containment     → Stop the bleeding without destroying evidence
4. Eradication     → Remove the threat from your environment
5. Recovery        → Restore systems to normal operation
6. Lessons Learned → Post-incident review and hardening
```

## Phase 1: Preparation

### Incident Classification

Define severity levels before an incident:

```yaml
# incident-classification.yml
severity:
  P1_critical:
    description: Active breach, data exfiltration, ransomware, complete service outage
    response_time: Immediate (< 15 minutes)
    escalation: CEO, CTO, Legal, on-call engineer
    examples:
      - Ransomware detected on production systems
      - Database containing PII accessed by unauthorized party
      - Production API serving attacker-controlled responses

  P2_high:
    description: Suspicious activity with potential for escalation
    response_time: < 1 hour
    escalation: Engineering lead, Security lead
    examples:
      - Multiple failed SSH login attempts from single IP
      - Unexpected admin account creation
      - Anomalous API usage patterns (data scraping)

  P3_medium:
    description: Policy violation, isolated vulnerability
    response_time: < 4 hours
    escalation: Engineering lead
    examples:
      - Credentials accidentally committed to public repo
      - Expired SSL certificate causing downtime
      - Vulnerability discovered in production dependency

  P4_low:
    description: Informational, no active threat
    response_time: Next business day
    escalation: Team lead
    examples:
      - Phishing email received (no click)
      - Port scan detected from external IP
```

### Contact List Template

```markdown
# Incident Response Contacts

## Internal
| Role | Name | Primary Phone | Secondary |
|------|------|--------------|-----------|
| Incident Commander | ... | ... | ... |
| Security Lead | ... | ... | ... |
| Engineering On-Call | PagerDuty | ... | ... |
| Legal Counsel | ... | ... | ... |
| Communications | ... | ... | ... |

## External
| Service | Contact | Notes |
|---------|---------|-------|
| Cloud Provider (AWS) | Support case + phone | Account ID: ... |
| Domain Registrar | ... | ... |
| Hosting Provider | ... | ... |
| Cyber Insurance | ... | Policy #... |
| Law Enforcement (FBI Cyber) | 1-855-292-3937 | IC3.gov |
```

### Tooling Checklist

```bash
# Tools to have ready before an incident

# Log analysis
apt install jq gawk grep sed

# Network forensics
apt install tcpdump wireshark-cli nmap netstat-nat

# Memory/disk forensics
apt install dc3dd volatility3

# File integrity checking
apt install aide tripwire

# Authentication log review
ausearch -k logins  # requires auditd
journalctl -u sshd --since "2 hours ago"

# Active connections snapshot
ss -tunap > /tmp/connections-$(date +%s).txt
netstat -anp > /tmp/netstat-$(date +%s).txt
```

## Phase 2: Identification

### Initial Triage Script

```bash
#!/bin/bash
# triage.sh — run immediately when an incident is suspected
# DO NOT run on a potentially compromised system without read-only mounts

TIMESTAMP=$(date +%Y%m%d-%H%M%S)
OUTDIR="/tmp/triage-${TIMESTAMP}"
mkdir -p "$OUTDIR"

echo "=== System State Snapshot: $TIMESTAMP ===" | tee "${OUTDIR}/summary.txt"

# Who is logged in
echo "--- Active sessions ---" >> "${OUTDIR}/summary.txt"
who >> "${OUTDIR}/summary.txt"
w >> "${OUTDIR}/summary.txt"

# Running processes
echo "--- Process list ---" >> "${OUTDIR}/summary.txt"
ps auxf >> "${OUTDIR}/summary.txt"

# Network connections
echo "--- Network connections ---" >> "${OUTDIR}/summary.txt"
ss -tunap >> "${OUTDIR}/summary.txt"

# Recently modified files (last 24 hours, excluding /proc and /sys)
echo "--- Files modified in last 24h ---" >> "${OUTDIR}/summary.txt"
find / -not -path "/proc/*" -not -path "/sys/*" -not -path "/dev/*" \
  -newer /tmp/.triage-marker -mtime -1 2>/dev/null \
  | head -200 >> "${OUTDIR}/summary.txt"

# Authentication log (last 100 events)
echo "--- Auth log (last 100) ---" >> "${OUTDIR}/summary.txt"
tail -100 /var/log/auth.log >> "${OUTDIR}/summary.txt"

# Cron jobs
echo "--- Crontabs ---" >> "${OUTDIR}/summary.txt"
for user in $(cut -f1 -d: /etc/passwd); do
  crontab -u "$user" -l 2>/dev/null && echo "[$user]" >> "${OUTDIR}/summary.txt"
done
ls /etc/cron* >> "${OUTDIR}/summary.txt"

# Loaded kernel modules
echo "--- Kernel modules ---" >> "${OUTDIR}/summary.txt"
lsmod >> "${OUTDIR}/summary.txt"

echo "Triage complete. Output: $OUTDIR"
tar czf "/tmp/triage-${TIMESTAMP}.tar.gz" "$OUTDIR"
```

## Phase 3: Containment

Containment stops the attack from spreading. The key tension is: **preserve evidence vs. stop damage**.

```bash
# Network isolation options (choose based on severity)

# Option 1: Block specific IP (low-impact, targeted)
sudo iptables -I INPUT -s 185.234.219.45 -j DROP
sudo iptables -I OUTPUT -d 185.234.219.45 -j DROP

# Option 2: Block all outbound except to your IR team's IP
sudo iptables -F
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -A INPUT -s YOUR_IR_TEAM_IP -j ACCEPT
sudo iptables -A OUTPUT -d YOUR_IR_TEAM_IP -j ACCEPT
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Option 3: Cloud — attach a deny-all security group (no reboot needed)
aws ec2 modify-instance-attribute \
  --instance-id i-1234567890abcdef0 \
  --groups sg-all-deny

# Take memory dump before rebooting (contains passwords, keys, malware in RAM)
sudo avml /external-drive/memory-$(date +%s).lime

# Snapshot disk before any changes
aws ec2 create-snapshot --volume-id vol-1234 --description "incident-$(date +%s)"
```

## Phase 4: Eradication

```bash
# Find persistence mechanisms

# Cron jobs added by attacker
grep -r "wget\|curl\|bash\|sh -i\|nc -e\|/tmp/" /etc/cron* /var/spool/cron/

# Systemd services installed recently
find /etc/systemd /usr/lib/systemd -name "*.service" -newer /tmp/.incident-baseline -ls

# SSH authorized keys added
find /root /home -name "authorized_keys" -newer /tmp/.incident-baseline

# SUID/SGID files (attackers sometimes add these for persistence)
find / -not -path "/proc/*" -not -path "/sys/*" \
  \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null

# Unusual listening ports
ss -tlnp | awk '{print $4, $6}' | sort

# Rootkit check
sudo rkhunter --check --skip-keypress
sudo chkrootkit
```

## Phase 5: Recovery

```bash
# Before bringing systems back online — verify integrity
sudo tripwire --check  # if Tripwire was installed before incident

# Rotate ALL credentials that could have been exposed
# - SSH keys on affected systems
# - API keys stored on the system
# - Database passwords
# - Service account tokens

# Rebuild from known-good image if rootkit suspected
# Never trust a compromised OS kernel

# Verify restored system behavior
curl https://api.ipify.org  # does it have outbound connectivity?
sudo ss -tlnp               # are only expected ports open?
sudo crontab -l             # no unexpected cron jobs?
```

## Phase 6: Lessons Learned

Run a post-incident review within 5 business days. The goal is systemic improvement, not blame.

```markdown
# Post-Incident Review Template

## Incident Summary
- Date/time detected:
- Date/time resolved:
- Duration:
- Severity:
- Systems affected:

## Timeline
| Time | Event | Who |
|------|-------|-----|
| HH:MM | First alert triggered | Monitoring system |
| HH:MM | On-call notified | PagerDuty |
| ... | ... | ... |

## Root Cause
What was the direct cause? (e.g., unpatched CVE-XXXX, leaked credential)
What allowed it to occur? (e.g., no patch management process, no secret rotation)

## What Went Well
-

## What Could Be Improved
-

## Action Items
| Action | Owner | Due Date | Priority |
|--------|-------|----------|----------|
| Deploy WAF rule to block exploit | Security | +3 days | High |
| Rotate all API keys quarterly | DevOps | +7 days | Medium |
| Add anomaly detection for auth | Engineering | +30 days | Medium |
```

## Related Reading

- [Threat Modeling Basics for Developers](/privacy-tools-guide/threat-modeling-basics-developers/)
- [How to Use OSSEC for Host Intrusion Detection](/privacy-tools-guide/ossec-host-intrusion-detection-setup/)
- [Lynis Linux Security Audit Guide](/privacy-tools-guide/lynis-linux-security-audit-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
