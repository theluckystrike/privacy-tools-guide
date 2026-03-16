---

layout: default
title: "How to Tell If Your Home Assistant or Alexa Was Compromised"
description: "A technical guide for developers and power users to detect signs of compromise in Home Assistant and Alexa devices. Learn to audit logs, identify suspicious patterns, and secure your smart home."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-tell-if-your-home-assistant-alexa-was-compromised/
categories: [security, smart-home, privacy]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Smart home devices like Home Assistant and Alexa have become central to our digital lives, controlling everything from lights and locks to climate systems and security cameras. When these platforms are compromised, attackers gain visibility into your daily routines and potentially control over physical access points. This guide provides developers and power users with practical techniques to detect whether your Home Assistant or Alexa has been compromised.

## Understanding the Attack Surface

Both Home Assistant and Alexa expose network interfaces that can be targeted. Home Assistant runs as a web server on your local network, accepting connections through its API and the frontend. Alexa communicates with Amazon's cloud, but the Alexa app and connected devices can be manipulated through account compromise or malicious skills.

The most common attack vectors include weak passwords, exposed APIs, compromised Alexa skills with excessive permissions, and network-level attacks on unencrypted communications.

## Detecting Compromise in Home Assistant

Home Assistant provides extensive logging that can reveal unauthorized access. The primary log file records events, authentication attempts, and component errors. Here's how to examine it programmatically:

```python
import requests
from datetime import datetime, timedelta

HA_URL = "http://homeassistant.local:8123"
TOKEN = "your_long_lived_access_token"

def get_auth_logs(hours=24):
    """Fetch authentication logs from Home Assistant"""
    end_time = datetime.now()
    start_time = end_time - timedelta(hours=hours)
    
    # Query the logbook API
    response = requests.get(
        f"{HA_URL}/api/logbook",
        headers={"Authorization": f"Bearer {TOKEN}"},
        params={
            "start_time": start_time.isoformat(),
            "end_time": end_time.isoformat()
        }
    )
    return response.json()

def analyze_failed_logins(events):
    """Identify suspicious login patterns"""
    failed_attempts = {}
    
    for event in events:
        if event.get("event_type") == "login_failed":
            source = event.get("data", {}).get("source")
            if source:
                failed_attempts[source] = failed_attempts.get(source, 0) + 1
    
    return failed_attempts

# Run analysis
logs = get_auth_logs(24)
failures = analyze_failed_logins(logs)

if failures:
    print("Potential brute force attempts detected:")
    for ip, count in failures.items():
        print(f"  {ip}: {count} failed attempts")
else:
    print("No failed login attempts in the last 24 hours")
```

This script queries the Home Assistant logbook API and identifies IP addresses with multiple failed authentication attempts. If you see repeated failures from an unfamiliar IP address, your instance may be under attack.

### Checking for Unauthorized Entities

Compromised installations often contain unfamiliar entities that you did not create. These might appear as automation, scripts, or new devices. Use the Home Assistant API to enumerate all entities and compare against a known-good baseline:

```python
def get_all_entities():
    """Retrieve all entities from Home Assistant"""
    response = requests.get(
        f"{HA_URL}/api/states",
        headers={"Authorization": f"Bearer {TOKEN}"}
    )
    return response.json()

def find_new_entities(current_entities, known_entities):
    """Identify entities not in the known baseline"""
    current_ids = {e["entity_id"] for e in current_entities}
    known_ids = set(known_entities)
    return current_ids - known_ids

# Load your baseline (store this securely)
known_entity_ids = []  # Load from saved baseline

all_entities = get_all_entities()
new_entities = find_new_entities(all_entities, known_entity_ids)

if new_entities:
    print("New entities detected:")
    for entity in sorted(new_entities):
        print(f"  {entity}")
```

Save the output of this script to create a baseline, then run it periodically to detect additions.

### Examining Automation Triggers

Malicious automations can execute arbitrary actions on your system. Review your automations for suspicious triggers, especially those that execute shell commands or make external HTTP requests:

```yaml
# Example of a suspicious automation pattern to search for
automation:
  - alias: "SUSPICIOUS - executes shell commands"
    trigger:
      - platform: state
        entity_id: binary_sensor.anything
    action:
      - service: shell_command.execute
        data:
          entity_id: shell_command.suspicious
```

Search your configuration files for automations containing `shell_command`, `http_request`, or `rest_command` services, as these can be used for data exfiltration or executing arbitrary code.

## Detecting Compromise in Alexa

Alexa compromise typically occurs at the account level rather than the device level. Amazon provides tools to review Alexa activity and connected skills.

### Reviewing Alexa Voice History

Amazon stores voice request history, which can reveal unauthorized commands. Navigate to the Alexa Privacy settings in the Amazon website or app, and review the "Voice History" section. Look for:

- Commands you did not issue
- Smart home device controls at unusual times
- Shopping orders you did not authorize

### Checking Connected Skills

Malicious skills can request excessive permissions and capture voice data. Access the Alexa app, go to Skills, and review "Your Skills." Remove any skills you do not recognize or that request unnecessary permissions:

```bash
# Use Amazon's Alexa API (requires developer credentials)
# List all enabled skills
aws alexaforbusiness list-skills --region us-east-1
```

Review each skill's permissions using the skill manifest. Skills requesting permissions beyond their stated purpose should be disabled and removed.

### Monitoring Alexa App Activity

The Alexa app shows a history of all voice requests and smart home actions. Check for patterns that indicate compromise:

1. Open the Alexa app
2. Navigate to Activity > Voice History
3. Filter by date range
4. Look for anomalies in timing and command types

## Network-Level Detection

For advanced users, monitoring network traffic can reveal compromise indicators. Both Home Assistant and Alexa should communicate only with expected endpoints.

### Identifying Unexpected Network Connections

Use tools like Wireshark or ngrep to monitor traffic from your smart home devices:

```bash
# Monitor HTTP traffic from Home Assistant
sudo ngrep -d en0 -i 'host 192.168.1.xxx' port 8123

# List established connections from Alexa devices
sudo lsof -i -P | grep -i alexa
```

Home Assistant should only connect to your local network and configured external services. Alexa devices communicate primarily with Amazon's AWS infrastructure. Any unexpected destination IP addresses warrant investigation.

### Setting Up Network Alerts

Create firewall rules that alert on unusual outbound connections:

```bash
# Create an iptables rule to log unexpected connections
sudo iptables -A OUTPUT -p tcp --dport 8123 -m state --state NEW \
  -m recent --set --name HA_CONNECTIONS

sudo iptables -A OUTPUT -p tcp --dport 8123 -m state --state NEW \
  -m recent --update --seconds 60 --hitcount 5 --name HA_CONNECTIONS \
  -j LOG --log-prefix "HA_BRUTE_FORCE: "
```

## Remediation Steps

If you detect signs of compromise, act immediately:

1. **Change passwords**: Update your Home Assistant password and any associated accounts. Use long, unique passwords stored in a password manager.

2. **Revoke API tokens**: In Home Assistant, generate new long-lived access tokens and invalidate old ones.

3. **Disable unknown entities**: Remove unfamiliar automations, scripts, and devices from your Home Assistant configuration.

4. **Factory reset Alexa devices**: If you suspect device-level compromise, perform a factory reset on affected Echo devices.

5. **Enable two-factor authentication**: Secure your Amazon account and any Home Assistant cloud integrations with 2FA.

6. **Update firmware**: Ensure all smart home devices run current firmware versions.

7. **Review network segmentation**: Isolate smart home devices on a separate VLAN to limit lateral movement in case of future compromise.

## Prevention Best Practices

Maintain ongoing security by implementing these practices:

- Use a dedicated admin account for Home Assistant, separate from daily use accounts
- Enable SSL/TLS for all Home Assistant connections
- Configure fail2ban to automatically block repeated authentication attempts
- Regularly audit your Alexa skills and remove unused ones
- Keep Home Assistant and all add-ons updated
- Use static IP assignments for critical devices to simplify monitoring

## Conclusion

Detecting compromise in smart home platforms requires vigilance and systematic monitoring. By regularly reviewing logs, tracking entity changes, monitoring network traffic, and maintaining security hygiene, you can identify threats before they cause significant damage. The techniques in this guide give developers and power users the tools to audit their installations and respond effectively to potential security incidents.

Implement these detection methods as part of your regular security routine, and maintain baseline documentation to quickly identify deviations from normal operation.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
