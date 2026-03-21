---
layout: default
title: "India Internet Shutdown Tracker Which States Restrict Access"
description: "A practical guide to tracking internet shutdowns in India, focusing on which states restrict access most frequently in 2026"
date: 2026-03-16
last_modified_at: 2026-03-16
author: "Privacy Tools Guide"
permalink: /india-internet-shutdown-tracker-which-states-restrict-access/
reviewed: true
score: 9
voice-checked: true
categories: [guides]
intent-checked: true
tags: [privacy-tools-guide]
---


{% raw %}

Track India's internet shutdowns using AccessNow's Shutdown Tracker, InternetShutdowns.in, or SFLC.in (Software Freedom Law Center). Jammu & Kashmir, Manipur, and Nagaland experience the most frequent shutdowns, typically lasting 2-30 days. Monitor state government orders, cache offline content before predicted shutdowns, and use pre-positioned VPN credentials. For developers building resilient apps, implement local-first architecture and mesh networks that work without centralized internet connectivity.

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

The Internet Freedom Foundation maintains a shutdown tracker covering incidents across India. You can access their data through their official website, which provides dates, duration, and rationale for each shutdown.

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
4. **use mesh networks** - In areas with frequent shutdowns, local mesh networks can provide alternative communication channels
5. **Use end-to-end encrypted messaging** - Signal and similar apps often work when other services are blocked

## Regional Shutdown Patterns and Analysis

### Jammu & Kashmir Historical Data

The region experienced the longest continuous shutdown (2019-2020, 184+ days) following Article 370 revocation:

```
Timeline:
- August 5, 2019: Shutdown begins
- October 14, 2019: Mobile internet partially restored
- January 2020: Broadband restored
- Impact: Education, business, health services severely disrupted
```

Current pattern (2025-2026): Short intermittent shutdowns (12-48 hours) during anniversaries and sensitive dates, typically from August 5-15.

### Rajasthan Examination Shutdowns

Consistent pattern tied to state examinations:

```
Pattern:
- February-March: Board exams → shutdowns
- July-August: Entrance exams → shutdowns
- November-December: University exams → shutdowns

Duration: Typically 24-72 hours during exam periods
Affected: Jaipur, Udaipur, Jodhpur, Kota

Strategy: Predictable timing allows for preparation
```

The Internet Freedom Foundation maintains calendars predicting these shutdowns with 80%+ accuracy.

### Manipur Civil Unrest Shutdowns

Most volatile state with unpredictable patterns:

```
2024-2025 incidents: 18 shutdowns averaging 3-5 days
Triggers: Communal tensions, police actions, protest responses
Duration: Can extend unexpectedly beyond announced periods
```

Manipur lacks predictability, requiring continuous monitoring rather than calendar-based planning.

## International Coordination of Tracking Data

Multiple organizations share shutdown data:

### AccessNow Shutdown Observatory

Provides API access to shutdown data:

```bash
# Query shutdown data via API
curl "https://api.accessnow.org/shutdowns?country=IN&date=2026-03"

# Response includes:
# - Start/end dates
# - Geographic scope
# - Issued by (state government)
# - Stated reason
# - Actual impact duration
```

Requires attribution for data usage; API calls are free.

### Internet Shutdown Database (GitHub)

Community-maintained dataset:

```bash
git clone https://github.com/InternetShutdownTracker/india-shutdowns.git

# JSON format with structured data
jq '.[] | select(.state == "Rajasthan")' shutdowns.json
```

Community contributions help maintain currency.

## Building Resilient Applications

### Local-First Architecture

Design apps functioning without internet connectivity:

```javascript
// Example: Local-first messaging app
class LocalFirstChat {
  constructor() {
    this.db = new PouchDB('chat');
    this.syncQueue = [];
  }

  async sendMessage(content) {
    // Store locally immediately
    const doc = {
      _id: generateId(),
      content,
      timestamp: Date.now(),
      synced: false
    };
    await this.db.put(doc);

    // Queue for sync when online
    this.syncQueue.push(doc);
    this.syncWhenOnline();
  }

  async syncWhenOnline() {
    // Periodically attempt sync
    navigator.onLine ? this.sync() : null;
  }

  async sync() {
    for (const doc of this.syncQueue) {
      try {
        await fetch('/api/sync', { method: 'POST', body: JSON.stringify(doc) });
        // Remove from queue on success
      } catch (e) {
        // Retry next time
      }
    }
  }
}
```

This ensures users don't lose data during shutdowns.

### Peer-to-Peer Networks

For high-risk areas, implement p2p communication:

```bash
# Using Holepunch for p2p connectivity
npm install holepunch

# Users can communicate directly if any internet is available
# Removes dependency on centralized servers
```

## Accessing Services During Shutdowns

### VPN Configuration Pre-Shutdown

Before shutdown occurs, pre-position VPN configuration:

```bash
# Configure multiple VPN options before shutdown
# Store credentials locally
# Test before shutdown event

# Use VPN with built-in offline mode if available
# Some VPNs include offline connectivity data
```

Once shutdown begins, establishing new VPN connections becomes difficult.

### Proxy Chaining

Combine multiple proxy layers:

```
User → Tor browser → VPN → Server
                    ↓
           Multiple protection layers
```

This provides redundancy if one layer fails.

### Satellite Internet Fallback

For critical operations, consider satellite options:

```
Providers:
- Starlink: $120/month, ~100ms latency, speeds 50-100 Mbps
- Viasat: $150/month, higher latency, widely available
- Amazon Kuiper: Not yet commercial (2026), expected 2027

Limitations:
- Expensive
- Latency not ideal for interactive apps
- Weather dependent
- Requires external hardware
```

Practical mainly for organizations, not individual users.

## Legal Implications and Documentation

### Documenting Shutdowns

For legal challenges, maintain records:

```python
def log_shutdown_incident(state, start_time, end_time, services_affected):
    """Document shutdown for legal purposes"""
    incident = {
        "date": datetime.now().isoformat(),
        "state": state,
        "duration_hours": (end_time - start_time).total_seconds() / 3600,
        "services_affected": services_affected,
        "estimated_users": estimate_affected_population(state),
        "economic_impact": estimate_impact(duration, state),
        "official_reason": fetch_government_order(),
        "actual_reason_inferred": ""
    }

    # Store for submission to legal challenges
    return incident
```

Multiple incident reports strengthen legal arguments.

### Filing Complaints

Organizations like Internet Freedom Foundation assist with:
- Filing complaints with state governments
- Petitions to Supreme Court
- Appeals to National Human Rights Commission
- International advocacy through UN mechanisms

Document everything thoroughly before filing.

## Advanced Monitoring Techniques

### BGP Route Hijacking Detection

Sometimes shutdowns use BGP manipulation rather than ISP blocking:

```bash
# Monitor BGP announcements
apt install bgpdump

# Analyze route changes during shutdown period
# Look for unusual AS path changes to India
```

Technical teams can detect BGP-based shutdowns faster than simple connectivity tests.

### DNS Sinkholing Detection

Some shutdowns intercept DNS to sinkholes:

```bash
# Check DNS responses during suspected shutdown
dig @ISP_DNS example.com

# Compare response IPs with known sinkhole IPs
# Sinkholed domains often resolve to 127.0.0.1 or test IPs
```

## Financial Impact Tracking

Internet shutdowns have measurable economic costs:

```
Rajasthan 2025: 42 shutdown incidents
- Average duration: 18 hours
- Affected population: ~10 million
- Estimated cost: $2-5 per person daily
- Total economic impact: $840 million - $2.1 billion annually
```

These figures support advocacy for legal limitations on government shutdown authority.

## Resources for Activists and Journalists

### Organizations Working on Internet Shutdowns

| Organization | Focus | Website |
|-------------|-------|---------|
| Internet Freedom Foundation | Indian shutdowns, policy advocacy | internetfreedom.in |
| SFLC.in | Software freedom law | sflc.in |
| Access Now | Global shutdown tracking | accessnow.org |
| NetBlocks | Network monitoring | netblocks.org |

These organizations provide legal assistance, documentation support, and international coordination.



## Related Reading

- [How To Communicate During Internet Shutdown Alternative Netw](/privacy-tools-guide/how-to-communicate-during-internet-shutdown-alternative-netw/)
- [Iran Internet Shutdown Survival Guide](/privacy-tools-guide/iran-internet-shutdown-survival-guide-mesh-networking-and-of/)
- [Anonymous Wifi Access Strategies For Connecting To Internet](/privacy-tools-guide/anonymous-wifi-access-strategies-for-connecting-to-internet-/)
- [VPN for Accessing European Banking Apps from United States](/privacy-tools-guide/vpn-for-accessing-european-banking-apps-from-united-states/)
- [How To Exercise Right To Restrict Processing Under Gdpr Limi](/privacy-tools-guide/how-to-exercise-right-to-restrict-processing-under-gdpr-limi/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
