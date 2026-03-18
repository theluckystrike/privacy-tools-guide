---

layout: default
title: "India Internet Shutdown Tracker: Which States Restrict Access Most Frequently (2026 Guide)"
description: "A practical guide to tracking internet shutdowns in India, focusing on which states restrict access most frequently in 2026."
date: 2026-03-16
author: theluckystrike
permalink: /india-internet-shutdown-tracker-which-states-restrict-access/
---

{% raw %}

India has experienced more internet shutdowns than any other country in recent years, with state governments frequently ordering restrictions on internet access for various reasons. For developers, journalists, and digital rights advocates, understanding when and where these shutdowns occur is essential for building resilient applications and documenting connectivity patterns.

This guide explores the tools and methods available for tracking internet shutdowns in India, with a specific focus on which states implement access restrictions most frequently in 2026.

## Understanding Internet Shutdowns in India

Internet shutdowns in India are typically ordered by state governments under Section 144 of the Code of Criminal Procedure, which authorizes authorities to restrict gatherings and communication. While the law was originally designed for public safety, it has increasingly been used to control internet access during protests, exams, and civil unrest.

The government classifies shutdowns as "pubic safety" or "national security" measures, though critics argue that many orders lack transparency and proper justification. As of 2026, the frequency and duration of shutdowns have made automated tracking a necessity for anyone relying on consistent internet connectivity.

## States with Most Frequent Internet Shutdowns

Based on data from organizations like the Software Freedom Law Centre (SFLC) and the Internet Freedom Foundation, several Indian states have consistently led in internet shutdown orders:

**Jammu and Kashmir** tops the list, with the region experiencing prolonged shutdowns following the revocation of Article 370 in 2019. While restrictions have eased somewhat, periodic shutdowns continue, particularly during anniversary dates and political events.

**Rajasthan** has seen a significant increase in shutdown orders, primarily during examination periods to prevent cheating and during local protests. The state recorded over 40 shutdown incidents in 2025 alone.

**Uttar Pradesh** orders shutdowns frequently during law and order situations, religious festivals, and exam periods. The state's large population means that each shutdown affects millions of users.

**Haryana** has implemented numerous restrictions, particularly in districts bordering Delhi during protest periods and examination seasons.

**Maharashtra** and **Madhya Pradesh** also feature prominently in shutdown records, with restrictions often tied to student examinations and communal tensions.

## Tools for Tracking Internet Shutdowns

Several platforms provide real-time data on internet shutdowns in India:

### 1. Internet Freedom Foundation's Internet Shutdown Tracker

The Internet Freedom Foundation maintains a comprehensive shutdown tracker covering incidents across India. You can access their data through their official website, which provides dates, duration, and rationale for each shutdown.

### 2. OONI Probe

The Open Observatory of Network Interference (OONI) provides a global network of testing probes that detect internet censorship. Running OONI Probe on your machine allows you to contribute to a global dataset while monitoring connectivity:

```bash
# Install OONI Probe CLI
pip install ooniprobe

# Run a basic connectivity test
ooni probe run --input https://example.com
```

### 3. NetBlocks

NetBlocks provides real-time network monitoring and tracks internet shutdowns globally. Their India-specific page offers visualizations of connectivity data and historical records.

### 4. GitHub-Based Trackers

Several developers maintain GitHub repositories that aggregate shutdown data:

```bash
# Example: Clone an Indian shutdown tracker repository
git clone https://github.com/priyankjair/india-internet-shutdowns.git
```

These repositories often include JSON files with structured data that you can parse programmatically:

```json
{
  "state": "Rajasthan",
  "date": "2026-01-15",
  "duration_hours": 72,
  "districts": ["Jaipur", "Jodhpur"],
  "reason": "Examination security"
}
```

## Building Your Own Shutdown Monitoring System

For developers who need real-time notifications, building a custom monitoring system is straightforward. Here's a basic approach using Python:

```python
import requests
import time
from datetime import datetime

def check_internet_connectivity():
    """Check if we can reach external hosts."""
    test_hosts = [
        "https://www.google.com",
        "https://www.cloudflare.com",
        "https://api.github.com"
    ]
    
    for host in test_hosts:
        try:
            response = requests.get(host, timeout=5)
            if response.status_code == 200:
                return True
        except requests.exceptions.RequestException:
            continue
    return False

def monitor_connection(interval=60):
    """Monitor connectivity and log interruptions."""
    log_file = "connectivity_log.txt"
    
    while True:
        timestamp = datetime.now().isoformat()
        is_connected = check_internet_connectivity()
        
        with open(log_file, "a") as f:
            status = "ONLINE" if is_connected else "OFFLINE"
            f.write(f"{timestamp}: {status}\n")
        
        if not is_connected:
            # Send notification (implement your preferred method)
            print(f"[ALERT] Connectivity lost at {timestamp}")
        
        time.sleep(interval)

if __name__ == "__main__":
    monitor_connection(interval=60)
```

This script logs connectivity status every minute, which helps identify patterns and durations of shutdowns affecting your location.

## Legal Framework and Your Rights

Understanding the legal framework surrounding internet shutdowns helps when documenting incidents or advocating for transparency. Under Indian law, shutdown orders must be reviewed by the relevant state government, but public disclosure of these orders varies significantly between states.

The Supreme Court has ruled that internet suspension is a violation of fundamental rights under Article 19 of the Constitution. Organizations like the Internet Freedom Foundation provide legal resources for those affected by unjustified shutdowns.

## Practical Recommendations

For developers and organizations that need resilience against internet shutdowns:

1. **Implement offline-first architectures** - Design applications that can function without continuous internet connectivity
2. **Use distributed DNS** - Services like Cloudflare or Google DNS can sometimes bypass restrictions
3. **Maintain documentation** - Record shutdown incidents with timestamps for potential legal challenges
4. **Leverage mesh networks** - In areas with frequent shutdowns, local mesh networks can provide alternative communication channels
5. **Use end-to-end encrypted messaging** - Signal and similar apps often work when other services are blocked

## Conclusion

Tracking internet shutdowns in India requires a combination of automated tools, community resources, and understanding of the legal landscape. While Jammu and Kashmir, Rajasthan, Uttar Pradesh, and Haryana lead in shutdown frequency, the pattern continues to evolve across all states.

For developers building applications for Indian users, incorporating shutdown detection and resilience into your systems is increasingly important. Resources like the OONI Probe, NetBlocks, and community-maintained GitHub repositories provide the data needed to understand and respond to these connectivity disruptions.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
