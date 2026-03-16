---

layout: default
title: "How to Tell If Your Home Assistant or Alexa Was Compromised: A Technical Guide"
description: "Learn the technical signs of Home Assistant and Alexa compromise. This guide covers log analysis, network monitoring, API key detection, and remediation for power users."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-tell-if-your-home-assistant-alexa-was-compromised/
categories: [security, privacy, smart-home]
reviewed: false
score: 0
intent-checked: false
voice-checked: false
---

{% raw %}

Smart home ecosystems like Home Assistant and Alexa have become central to modern home automation. These platforms process sensitive data—voice commands, automation schedules, and integration with locks, cameras, and sensors. When compromised, an attacker gains persistent access to your daily routines, physical security systems, and personal conversations. This guide provides developers and power users with technical methods to detect compromise in Home Assistant and Alexa environments.

## Understanding the Attack Surface

Both Home Assistant and Alexa expose multiple attack vectors. Home Assistant, being self-hosted, can be compromised through exposed web interfaces, weak authentication, vulnerable integrations, or supply chain attacks on custom components. Alexa devices face risks from unauthorized Alexa Skills, account Takeover through credential stuffing, or voice recordings accessed via compromised Amazon accounts.

The key difference is visibility: Home Assistant gives you direct access to logs and system state, while Alexa requires you to rely on Amazon's limited activity dashboards and API access.

## Detecting Home Assistant Compromise

### Review Authentication Logs

Home Assistant logs all authentication attempts. Access these through the UI or by examining the log files directly:

```bash
# Access Home Assistant logs via CLI (if SSH is enabled)
ha logs --container core | grep -i "auth"
```

Look for these warning signs:

- Successful logins from unfamiliar IP addresses
- Multiple failed authentication attempts followed by a success
- Sessions active at times you weren't using the system
- API tokens you don't recognize

To list all active sessions in Home Assistant 2023.8 and later:

```python
# Using Home Assistant API
import hassapi as hass
import json

@state_trigger("True")
def list_sessions(event):
    # This requires the homeassistant.api component
    # and appropriate permissions
    print("Check UI: Profile -> Sessions for active tokens")
```

The safer approach is through the UI: navigate to your profile (top-right avatar), scroll to "Long-Lived Access Tokens," and revoke any tokens you did not create.

### Analyze Automation and Script Modifications

Compromised systems often introduce unauthorized automations or modify existing ones. Export your configuration:

```bash
# Backup automations.yaml
cp ~/.homeassistant/automations.yaml automations_backup_$(date +%Y%m%d).yaml
```

Review the file for unfamiliar automations—particularly those that:

- Trigger on unusual events (device state changes you don't recognize)
- Execute HTTP requests to unknown domains
- Run shell commands or system commands
- Enable or disable your alarm system, locks, or cameras

Compare against your known-good backup:

```bash
# Diff against known good backup
diff automations_backup_known_good.yaml automations_current.yaml
```

### Check for Unauthorized Integrations

Malicious integrations can provide backdoor access. Navigate to Settings → Devices & Services → Integrations and verify every integration. Remove any that you didn't intentionally add.

For programmatic verification, query the integration registry:

```bash
# Get list of installed integrations via HA CLI
ha core integrations | grep -v "^#"
```

### Monitor Network Traffic

Network-level monitoring catches compromise that hides at the application layer. Set up packet capture on your Home Assistant host:

```bash
# Capture HTTP traffic (requires root)
tcpdump -i eth0 -w ha_traffic.pcap port 80 or port 443
```

Analyze the capture for connections to suspicious IP addresses, unusual domains, or data exfiltration patterns. Tools like Wireshark or Zeek can parse these exports for IOCs (Indicators of Compromise).

Alternatively, use a local DNS resolver like Pi-hole to log DNS queries from your Home Assistant instance. Any DNS queries to unknown domains warrant investigation.

## Detecting Alexa Compromise

### Review Alexa Activity History

Amazon provides limited activity logging. Open the Alexa app and navigate to Activity to review:

- Voice command history (look for commands you didn't issue)
- Smart home device actions (check for unauthorized device controls)
- Skills enabled (verify each skill is intentional)
- Shopping activity (unauthorized purchases are a red flag)

### Check Alexa Voice History

Amazon stores voice recordings. Access them through:

1. Go to amazon.com/hamm
2. Navigate to "Privacy Settings" → "Voice Recordings"
3. Review recordings for unfamiliar commands

Attackers with account access can issue voice commands to unlock doors, disarm security systems, or retrieve personal information.

### Verify Linked Accounts

Compromised Alexa skills can request account linking to gain access to your Amazon account or third-party services. Check linked accounts:

- Open Alexa app → Settings → Account Settings → Linked Accounts
- Remove any unauthorized links

### Enable Alexa Guard Notifications

Alexa Guard (if enabled) can detect unusual activity. Configure Guard to send alerts for:

- Detected sounds (glass breaking, smoke alarms)
- Smart home device activity when you're away

Any alerts you don't recognize could indicate compromise.

## Cross-Platform Indicators

Some signs of compromise appear regardless of platform:

### Unexpected Device Behavior

- Lights turning on or off at unusual times
- Thermostat settings changing without input
- Smart locks engaging or disengaging spontaneously
- Cameras moving or recording unexpectedly
- Speaker devices activating spontaneously (listening indicators lighting up)

### Network Anomalies

- New devices on your network, particularly those with manufacturer IDs you don't recognize
- Unusual bandwidth consumption
- New DHCP reservations or static IP assignments

### API Key Exposure

Developer integrations with Home Assistant often use long-lived access tokens. If you've accidentally committed these to version control:

```bash
# Search for exposed tokens in git history
git log --all -p -S "eyJ" --source -- . 2>/dev/null | grep -i "token\|api"
```

The `eyJ` pattern indicates JWT tokens. Rotate any exposed credentials immediately.

## Remediation Steps

If you detect compromise, act quickly:

1. **Disconnect from the network**: Isolate affected devices by removing them from WiFi or disabling their network access
2. **Revoke all tokens and API keys**: In Home Assistant, delete all Long-Lived Access Tokens. In Alexa, change your Amazon password and enable two-factor authentication
3. **Factory reset devices**: For Alexa Echo devices, hold the Reset button. For Home Assistant, reinstall the OS from a verified source
4. **Update credentials**: Change passwords for any accounts linked to your smart home (Google, Amazon, IFTTT, etc.)
5. **Re-verify integrations**: Re-add only trusted integrations from known sources
6. **Enable two-factor authentication**: On both Amazon accounts and Home Assistant (use TOTP-based 2FA)

## Prevention Strategies

Preventing compromise requires defense in depth:

- **Network segmentation**: Place smart home devices on a separate VLAN from computers and servers
- **Regular audits**: Schedule monthly reviews of active sessions, automations, and integrations
- **Minimize cloud dependencies**: Where possible, run Home Assistant without cloud connectivity (disable cloud hooks)
- **Use local-only integrations**: Prefer integrations that don't require cloud APIs
- **Monitor continuously**: Set up automated alerting for unusual activity using tools like n8n or Node-RED connected to your Home Assistant instance

## Conclusion

Detecting compromise in smart home ecosystems requires proactive monitoring. Home Assistant's self-hosted nature provides log access and system transparency that Alexa cannot match. For both platforms, regular audits of authentication events, automation rules, and network traffic form the foundation of security. When in doubt, rotate credentials and factory reset—prevention is far easier than remediation.

The responsibility for smart home security ultimately rests with the owner. By implementing the detection methods and prevention strategies outlined here, you significantly reduce the risk of unauthorized access to your connected home.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
