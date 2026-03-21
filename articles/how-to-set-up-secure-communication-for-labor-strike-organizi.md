---
layout: default
title: "Set Up Secure Communication for Labor Strike Organizing"
description: "A practical guide for developers and power users setting up secure communication infrastructure for labor strike organizing. Covers encrypted"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-secure-communication-for-labor-strike-organizing/
categories: [guides, security]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

To set up secure labor strike communication, deploy Signal for primary encrypted messaging with disappearing messages enabled, Session Messenger for decentralized backup, and self-hosted Jitsi Meet for group calls. Layer in encrypted email through ProtonMail, use a separate device or virtual machine, implement dead man's switches for credential management, and establish clear operational security protocols before coordinating sensitive strike activities.

## Threat Modeling for Strike Communications

Before deploying any communication tools, establish a clear threat model. Strike communications face specific risks:

1. **Employer surveillance** — Management may monitor workplace devices, networks, and communications
2. **Legal discovery** — Communications may be subpoenaed during labor disputes
3. **Infiltration** — Bad actors may attempt to join organizing efforts to gather intelligence
4. **Device seizure** — Phones and computers may be confiscated during pickets or arrests
5. **Service disruption** — Employers or authorities may attempt to block or jam communication channels

Your threat model determines which tools and configurations are appropriate. No solution provides absolute security, but layered defenses significantly reduce your attack surface.

## Core Communication Stack

### Primary Channel: Signal

Signal provides the strongest balance of security and usability for real-time coordination. Its implementation of the Signal protocol offers:

- End-to-end encryption by default
- Sealed sender technology hiding metadata from servers
- Disappearing messages for sensitive discussions
- Registration lock preventing SIM-swap-based account takeover

For strike organizing, configure Signal with these settings:

1. Enable **disappearing messages** (24-72 hours recommended)
2. Enable **screen lock** requiring biometric or PIN access
3. Enable **registration lock** in settings
4. Use **group chats** rather than broadcast lists for coordination
5. Verify **safety numbers** with core organizing committee members

### Secondary Channel: Session Messenger

Session provides an alternative that operates without phone numbers, routing traffic through onion nodes for metadata resistance. This matters when phone numbers could link participants to their real identities.

Session operates on the Signal protocol but routes through LokiNET, providing:
- No phone number requirement for account creation
- Decentralized onion routing hiding communication metadata
- Local-only message storage with no server-side history

Deploy Session as a backup channel when Signal becomes unavailable or when participants need additional anonymity.

### Offline Coordination: Briar

When internet access becomes restricted—whether through network blocks, shutdowns, or physical displacement—Briar provides mesh networking via Bluetooth and Wi-Fi Direct. Devices can synchronize messages when within range without any internet connection.

Briar deployment strategy:
1. Install Briar on Android devices (iOS version in development)
2. Establish trusted device pairs before operational need arises
3. Create dedicated "contact groups" for different organizing functions
4. Designate runners who can physically carry messages between locations

## Self-Hosted Infrastructure

For organizations wanting greater control, self-hosted options provide infrastructure independence.

### Matrix with Element

Matrix offers federated messaging with end-to-end encryption capability. Deploy Synapse homeserver for organizational control:

```bash
# docker-compose.yml for Synapse
version: '3'
services:
  synapse:
    image: matrixdotorg/synapse:latest
    container_name: synapse
    volumes:
      - ./synapse:/data
    ports:
      - "8008:8008"
      - "8448:8448"
    environment:
      - SYNAPSE_SERVER_NAME=organize.yourdomain.org
      - SYNAPSE_REPORT_STATS=no
    restart: unless-stopped
```

Generate configuration with:
```bash
docker run -it --rm -v ./synapse:/data \
  -e SYNAPSE_SERVER_NAME=organize.yourdomain.org \
  matrixdotorg/synapse:latest generate
```

Configure Element client for sensitive rooms:
- Enable end-to-end encryption in room settings
- Set history visibility to "members only since joining"
- Disable guest access entirely
- Implement key rotation via admin API:

```json
POST /_matrix/client/r0/admin/rooms/{room_id}/state/m.room.encryption
{
  "algorithm": "m.megolm.v1.aes-sha2",
  "rotation_period_ms": 604800000,
  "rotation_period_msgs": 100
}
```

### Encrypted Video: Jitsi Self-Hosted

For video coordination meetings, self-hosted Jitsi provides encrypted calls without relying on third-party services:

```bash
# Jitsi docker deployment
version: '3'
services:
  jitsi:
    image: jitsi/web
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./jitsi:/config
      - ./jitsi/transcripts:/var/lib/jitsi/transcripts
    environment:
      - ENABLE_AUTH=1
      - ENABLE_LOBBY=1
      - ENABLE_END_TO_END_ENCRYPTION=1
```

Configure lobby mode to require explicit admission for sensitive meetings.

## Operational Security Practices

Technical tools require operational discipline to be effective.

### Device Security

- Use dedicated devices for organizing communications when possible
- Enable full-disk encryption
- Configure secure boot with custom keys
- Use alphanumeric PINs rather than biometrics for sensitive situations (biometrics can be compelled)
- Enable Find My Device / remote wipe capability

### Network Considerations

- Use VPN when accessing organizing communications over employer or public networks
- Avoid organizing discussions on workplace WiFi
- Consider Tor Browser for accessing communication services in high-risk environments
- Disable WiFi auto-join to prevent inadvertent network connections

### Communication Discipline

- Use code words for sensitive operational details
- Rotate encryption keys periodically
- Verify identities through out-of-band channels
- Assume all non-encrypted communications may be intercepted
- Delete sensitive messages after action items are completed

## Backup and Recovery

Establish recovery procedures before operational need:

1. **Signal**: Print backup code and store in secure offline location
2. **Matrix**: Export 4S recovery key, store offline
3. **Session**: Record recovery phrase in secure location
4. **All**: Designate trusted contacts for emergency access



## Related Articles

- [Set Up Secure Communication For Labor Strike Organizing](/privacy-tools-guide/how-to-set-up-secure-communication-for-labor-strike-organizing/)
- [Lawyer Client Privilege Digital Communication Secure Setup C](/privacy-tools-guide/lawyer-client-privilege-digital-communication-secure-setup-c/)
- [Secure Communication Plan Template for Organizations.](/privacy-tools-guide/secure-communication-plan-template-for-organizations-handlin/)
- [Turkey Secure Communication Guide For Activists And Ngos Ope](/privacy-tools-guide/turkey-secure-communication-guide-for-activists-and-ngos-ope/)
- [How to Set Up Encrypted Communication for Mutual Aid Network](/privacy-tools-guide/how-to-set-up-encrypted-communication-for-mutual-aid-network/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
