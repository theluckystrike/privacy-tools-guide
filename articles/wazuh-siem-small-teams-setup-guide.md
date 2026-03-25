---
layout: default
title: "How to Set Up Wazuh SIEM for Small Teams"
description: "Deploy Wazuh all-in-one SIEM on a single Linux server to collect logs, detect threats, run compliance checks, and alert your team without enterprise pri..."
date: 2026-03-22
author: theluckystrike
permalink: /wazuh-siem-small-teams-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
How to Set Up Wazuh SIEM for Small Teams

Wazuh is a free, open-source security platform that combines log management, intrusion detection, vulnerability scanning, and compliance checks. For small teams, the all-in-one installer gets you a full SIEM stack. Wazuh Manager, Indexer (OpenSearch), and Dashboard. on a single server in under 30 minutes.

Hardware Requirements

| Component | Minimum | Recommended |
|---|---|---|
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 50 GB SSD | 200 GB SSD |
| OS | Ubuntu 22.04/24.04 or RHEL 8/9 | Ubuntu 24.04 |

For a 10-agent deployment monitoring a small team's infrastructure, 8 GB RAM is workable but tight. Scale up if you're ingesting high-volume logs from Zeek or a web server.

---

1. All-in-One Installation

Wazuh provides an official installation assistant that handles certificates, configuration, and service setup:

```bash
Download and verify the installer
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh.sha512

sha512sum -c wazuh-install.sh.sha512

Run all-in-one installation (takes 10-15 minutes)
sudo bash wazuh-install.sh --all-in-one -v

Save the admin password from the final output. you'll need it for the Dashboard
```

This installs:
- Wazuh Indexer (OpenSearch) on port 9200
- Wazuh Manager on port 1514 (agent communication) and 55000 (API)
- Wazuh Dashboard (Kibana-based UI) on port 443

---

2. Access the Dashboard

Open `https://<your-server-ip>` in a browser. Accept the self-signed certificate warning and log in with:
- Username: `admin`
- Password - (printed at the end of installation, also retrievable with `sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt`)

---

3. Install the Wazuh Agent on Linux Endpoints

Run this on each server you want to monitor:

```bash
Replace WAZUH_MANAGER with the IP of your Wazuh server
WAZUH_MANAGER="10.0.1.50" WAZUH_AGENT_NAME="web-server-1" \
  bash <(curl -s https://packages.wazuh.com/4.9/wazuh-agent-install.sh)

sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
sudo systemctl status wazuh-agent
```

On Windows (PowerShell, run as admin):

```powershell
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.9/windows/wazuh-agent-4.9.0-1.msi" `
  -OutFile "${env:TEMP}\wazuh-agent.msi"
msiexec /i "${env:TEMP}\wazuh-agent.msi" WAZUH_MANAGER="10.0.1.50" `
  WAZUH_AGENT_NAME="windows-laptop-1" /quiet
net start WazuhSvc
```

---

4. Verify Agent Connection

On the Wazuh Manager:

```bash
List connected agents
sudo /var/ossec/bin/agent_control -la

Check agent status by ID
sudo /var/ossec/bin/agent_control -s -u 001
```

Or in the Dashboard - Agents > verify the agent appears and shows "Active".

---

5. Configure Log Collection

Wazuh reads syslog, application logs, and Windows Event Logs automatically. Add custom log sources in the agent's `ossec.conf`:

```bash
sudo tee -a /var/ossec/etc/ossec.conf > /dev/null <<'EOF'
<ossec_config>
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/nginx/access.log</location>
  </localfile>

  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/nginx/error.log</location>
  </localfile>

  <localfile>
    <log_format>json</log_format>
    <location>/var/log/myapp/app.log</location>
  </localfile>
</ossec_config>
EOF

sudo systemctl restart wazuh-agent
```

---

6. Enable Vulnerability Scanning

Wazuh can scan agents for known CVEs using the installed package list:

On the Manager, edit `/var/ossec/etc/ossec.conf`:

```xml
<vulnerability-detector>
  <enabled>yes</enabled>
  <interval>12h</interval>
  <min_full_scan_interval>6h</min_full_scan_interval>
  <run_on_start>yes</run_on_start>

  <provider name="canonical">
    <enabled>yes</enabled>
    <os>focal</os>
    <os>jammy</os>
    <os>noble</os>
    <update_interval>1h</update_interval>
  </provider>

  <provider name="redhat">
    <enabled>yes</enabled>
    <update_interval>1h</update_interval>
  </provider>
</vulnerability-detector>
```

```bash
sudo systemctl restart wazuh-manager
```

Results appear in Dashboard > Vulnerability Detection.

---

7. Write a Custom Detection Rule

Wazuh ships with 2000+ rules. Add your own in `/var/ossec/etc/rules/local_rules.xml`:

```xml
<!-- Alert when a new user is added to the sudo group -->
<group name="local,adduser,sudo">
  <rule id="100001" level="10">
    <if_sid>5901</if_sid>
    <match>sudo</match>
    <description>User added to sudo group</description>
    <mitre>
      <id>T1098</id>
    </mitre>
  </rule>

  <!-- Alert on failed sudo attempts (possible privilege escalation probe) -->
  <rule id="100002" level="7" frequency="5" timeframe="60">
    <if_matched_sid>5402</if_matched_sid>
    <description>Multiple failed sudo attempts from same user</description>
  </rule>
</group>
```

```bash
Test and reload rules without restarting
sudo /var/ossec/bin/wazuh-logtest
sudo systemctl reload wazuh-manager
```

---

8. Configure Active Response

Active response lets Wazuh automatically block IPs that trigger certain rules. similar to Fail2Ban but managed centrally:

In `/var/ossec/etc/ossec.conf` on the Manager:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5712</rules_id>   <!-- SSH brute force -->
  <timeout>600</timeout>       <!-- Block for 10 minutes -->
</active-response>

<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>31151</rules_id>   <!-- Web scan detected -->
  <timeout>1800</timeout>
</active-response>
```

The `firewall-drop` command uses `iptables` or `nftables` depending on what's available on the agent.

---

9. Set Up Email Alerts

```xml
<!-- In Manager ossec.conf -->
<global>
  <email_notification>yes</email_notification>
  <smtp_server>smtp.example.com</smtp_server>
  <email_from>wazuh@example.com</email_from>
  <email_to>security-team@example.com</email_to>
  <email_maxperhour>50</email_maxperhour>
  <email_alert_level>7</email_alert_level>   <!-- Only severity 7+ -->
</global>
```

For Slack, use the Wazuh integration framework:

```bash
sudo tee /var/ossec/integrations/custom-slack.py > /dev/null << 'PYEOF'
Use the wazuh-slack integration package from wazuh.com/resources
PYEOF

Or use the built-in webhook integration
```

---

10. CIS Benchmark Compliance

Wazuh ships with OpenSCAP and SCA (Security Configuration Assessment) checks:

```bash
Check SCA results
sudo /var/ossec/bin/wazuh-logtest < /dev/null 2>&1 | head

View SCA policy results in Dashboard: Security Configuration Assessment
Each check shows Pass/Fail with remediation steps
```

Dashboard > Security Configuration Assessment shows a scored compliance report for CIS Level 1 and Level 2 benchmarks.

---

Maintenance

```bash
Check indexer disk usage
curl -sk -u admin:<password> https://localhost:9200/_cat/indices?v

Delete old indices (retention management)
In Dashboard - Index Management > Indices > filter by date > delete

Update Wazuh
sudo apt upgrade wazuh-manager wazuh-indexer wazuh-dashboard
```

---

Related Reading

- [How to Set Up Snort IDS on Linux](/snort-ids-linux-setup-guide/)
- [How to Use Zeek for Network Monitoring](/zeek-network-monitoring-guide/)
- [How to Set Up OpenSCAP for Compliance Scanning](/openscap-compliance-scanning-setup-guide/)
- [How to Set Up age Encryption for Teams](/age-encryption-team-setup-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://bestremotetools.com/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)

---

Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
