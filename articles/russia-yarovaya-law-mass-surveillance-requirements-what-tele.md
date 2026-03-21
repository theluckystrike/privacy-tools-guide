---
layout: default
title: "Russia Yarovaya Law Mass Surveillance Requirements What Tele"
description: "Russia Yarovaya Law: What Data Telecom Companies Must. — privacy guide covering tools, techniques, and best practices to protect your data and digital"
date: 2026-03-16
author: theluckystrike
permalink: /russia-yarovaya-law-mass-surveillance-requirements-what-tele/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

The Yarovaya Law (Федеральный закон № 374-ФЗ), officially known as the "Yarovaya package," is a sweeping Russian surveillance legislation enacted in 2016 that fundamentally changed data retention requirements for telecom operators, internet service providers, and messaging services. For developers and power users working with Russian infrastructure or serving Russian users, understanding these requirements is essential for compliance and privacy-conscious architecture decisions.

## What is the Yarovaya Law?

Named after Russian parliamentarian Irina Yarovaya who sponsored the legislation, Federal Law No. 374-FZ was adopted in July 2016 and subsequently amended multiple times. The law dramatically expanded Russia's surveillance capabilities by mandating extensive data retention and providing law enforcement agencies with unprecedented access to user communications data.

The legislation applies to:
- Telecom operators (mobile and fixed-line)
- Internet service providers (ISPs)
- Messaging services and VoIP providers
- VPN and proxy services
- Email providers
- Data centers and hosting providers

## Mandatory Data Retention Requirements

### Telecom Operator Obligations

Russian telecom operators must store the following data:

```json
{
  "voice_calls": {
    "caller_number": "required - stored for 6 years",
    "recipient_number": "required - stored for 6 years", 
    "call_duration": "required - stored for 6 years",
    "imei_numbers": "required - stored for 6 years",
    "imsi_numbers": "required - stored for 6 years",
    "cell_tower_location": "required - stored for 6 years",
    "timestamp": "required - stored for 6 years"
  },
  "sms_messages": {
    "sender_number": "required - stored for 6 years",
    "recipient_number": "required - stored for 6 years",
    "message_content": "required - stored for 6 years",
    "timestamp": "required - stored for 6 years"
  },
  "data_services": {
    "ip_addresses_assigned": "required - stored for 6 years",
    "traffic_metadata": "required - stored for 6 years",
    "session_duration": "required - stored for 6 years"
  }
}
```

### Internet Service Provider Requirements

ISPs operating in Russia must retain:

```python
# ISP data retention requirements under Yarovaya Law
ISP_RETENTION_REQUIREMENTS = {
    "subscriber_data": [
        "full_name",           # Full legal name
        "passport_details",    # Government ID
        "registration_address", # Legal address
        "installation_address", # Service location
        "contract_start_date"
    ],
    "network_data": [
        "assigned_ip_address",     # Dynamic/static IP
        "connection_timestamp",
        "session_duration", 
        "volume_of_data_transferred",
        "protocol_information",
        "destination_ip_addresses",
        "dns_queries_made"
    ],
    "access_logs": [
        "authentication_records",
        "service_usage_times",
        "billing_information"
    ],
    "retention_period": "6 years"
}
```

### Messaging and VoIP Services

The Yarovaya Law significantly impacts messaging applications:

- **All messages must be traceable**: Even encrypted services must maintain the ability to provide message metadata to authorities
- **User identification required**: Services must verify user identity through Russian phone numbers
- **Real-time interception capability**: Providers must build systems allowing live surveillance
- **Content storage**: Message content may need to be stored for up to 1 year

## Technical Implementation Examples

### Log Format for ISP Compliance

Here's how an ISP might structure compliance logs:

```javascript
// Compliance log structure for Yarovaya Law
const complianceLog = {
  event_type: "SESSION_START",
  timestamp: "2026-03-16T10:30:00Z",
  subscriber: {
    account_id: "ACC-123456",
    legal_name: "Ivan Petrov",
    passport: "12 34 567890",
    address: "Moscow, Russia"
  },
  network_session: {
    session_id: "SES-7890123456",
    assigned_ip: "85.11.22.33",
    nas_ip: "10.0.0.1",
    connection_type: "PPPoE",
    vlan_id: 100,
    ingress_interface: "eth0/0/1"
  },
  metadata: {
    destination_ips: ["93.184.216.34"],
    destination_domains: ["example.com"],
    protocol: "TCP",
    port: 443,
    bytes_sent: 15234,
    bytes_received: 45678
  }
};
```

### TLS Certificate Infrastructure Implications

The Yarovaya Law has significant implications for TLS/SSL certificate infrastructure:

```bash
# Russian CA requirements under Yarovaya Law
# TLS certificates must be issueable by FSB-approved CAs
# Service providers must be able to decrypt TLS traffic

# Required configuration for compliance:
1. Deploy FSB-approved SSL/TLS interception proxies
2. Maintain private keys accessible to authorities
3. Implement SORM-compliant logging systems
4. Configure packet capture for all HTTPS traffic
5. Store SSL session keys for decryption capability
```

## VPN and Proxy Service Requirements

VPN providers operating in Russia face particularly stringent requirements:

| Requirement | Details |
|------------|---------|
| License | Must obtain government license to operate |
| User Logging | Store all connection logs for 1 year |
| IP Assignment | Record which IP assigned to which user |
| Timestamps | Log connection start/end times |
| Bandwidth | Record volume of data transferred |
| Metadata | Store all connection metadata |
| Access | Provide real-time access to FSB |

```python
# VPN provider compliance data structure
VPN_COMPLIANCE_DATA = {
    "connection_logs": {
        "user_id": "required",
        "real_ip_address": "required",
        "vpn_ip_assigned": "required",
        "connection_start": "required - ISO 8601",
        "connection_end": "required - ISO 8601",
        "bytes_transmitted": "required",
        "destination_servers": "required",
        "protocol_used": "required"
    },
    "retention_period": "1 year minimum",
    "access_requirement": "Real-time FSB access capability"
}
```

## Implications for Developers and Power Users

### For Application Developers

If you're building services for Russian users:

1. **Infrastructure planning**: Plan for data retention systems from the start
2. **Data minimization**: Collect only necessary data to limit liability
3. **Encryption considerations**: Understand that encryption may not exempt you from retention requirements
4. **User notifications**: Implement clear data handling disclosures
5. **Jurisdictional awareness**: Consider where you host data and how laws apply cross-border

### For Privacy-Conscious Users

Users concerned about these requirements can:

1. **Use foreign services**: Services outside Russian jurisdiction may have different obligations
2. **End-to-end encryption**: Use applications with genuine E2E encryption (though metadata may still be collected)
3. **Tor and I2P**: These networks provide stronger anonymity guarantees
4. **VPN considerations**: Understand that Russian-licensed VPNs must log data
5. **Device security**: Use devices with hardware security features to protect local data

### Technical Countermeasures

For users implementing technical countermeasures:

```bash
# Network isolation approach
# Route sensitive traffic through non-Russian exit nodes

# DNS configuration
# Use DNS over HTTPS with non-Russian providers
# Example: Cloudflare (1.1.1.1) or Google (8.8.8.8)

# Traffic analysis prevention
# Use obfuscated protocols (obfs4, Meek, Shadowsocks)
# Implement perfect forward secrecy in custom applications
```

## Enforcement and Penalties

Non-compliance with Yarovaya Law requirements carries significant penalties:

- **Fines**: Up to 1 million rubles for operators
- **License revocation**: ISPs and telcos can lose operating licenses
- **Criminal liability**: executives can face criminal charges
- **Website blocking**: Non-compliant services can be blocked in Russia

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
