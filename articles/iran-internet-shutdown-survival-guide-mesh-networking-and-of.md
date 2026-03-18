---

layout: default
title: "Iran Internet Shutdown Survival Guide: Mesh Networking."
description: "A technical guide to mesh networking and offline communication tools for maintaining connectivity during internet shutdowns. Includes practical setup."
date: 2026-03-16
author: "theluckystrike"
permalink: /iran-internet-shutdown-survival-guide-mesh-networking-and-of/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
---

# Iran Internet Shutdown Survival Guide: Mesh Networking and Offline Communication Tools 2026

When internet shutdowns occur in Iran, mesh networking and offline communication tools enable connectivity without relying on centralized infrastructure. Partial mesh networks using Wi-Fi Direct and Bluetooth Low Energy offer the best practical balance, with applications like Briar providing encrypted peer-to-peer messaging that syncs automatically when devices reconnect. For developers and technical users, understanding these decentralized technologies is essential for maintaining communication during network disruptions.

## Understanding Mesh Networking Fundamentals

Mesh networking creates interconnected networks where each node can communicate directly with nearby nodes, forming a resilient web of communication that doesn't require internet access. Unlike traditional client-server models, mesh networks are decentralized—no single point of failure can bring down the entire system.

There are three primary mesh networking topologies:

- **Full mesh**: Every node connects to every other node (impractical for large networks)
- **Partial mesh**: Nodes connect to multiple nearby nodes but not all (optimal balance)
- **Hybrid mesh**: Combines mesh principles with traditional infrastructure when available

For practical emergency communication, partial mesh networks using Wi-Fi Direct and Bluetooth Low Energy (BLE) offer the best balance of range, battery efficiency, and device compatibility.

## Briar: Mesh Messaging for Android

Briar stands as the most mature offline mesh messaging application available. It creates encrypted peer-to-peer connections over Bluetooth, Wi-Fi Direct, and Tor hidden services, enabling communication without any internet connectivity.

### Installation and Setup

```bash
# Briar is available on F-Droid and Google Play Store
# No command-line installation required for Android

# Key features:
# - End-to-end encryption via Signal protocol
# - No phone number requirement (contacts via QR code exchange)
# - Forums and groups work offline
# - Message syncs automatically when peers connect
```

The application operates entirely without servers—messages propagate through the mesh when devices are in range. This makes it ideal for protest coordination where participants can relay messages across distances without direct connectivity to all participants.

### Practical Deployment

For community organizers, the recommended approach involves:

1. Pre-distributing the APK through secure channels (encrypted USB drives)
2. Exchanging contact codes in person before events
3. Establishing "message courier" nodes at strategic locations
4. Using forums for broadcast announcements to all connected peers

## Serval Mesh: Mesh Networking for Multiple Platforms

Serval Mesh offers cross-platform support including Android, iOS, and dedicated mesh hardware. Unlike Briar's focus on messaging, Serval emphasizes voice communication and file sharing over mesh networks.

### Mesh Potato: Dedicated Hardware

The Mesh Potato project combines mesh networking with voice capability in a dedicated device. While primarily designed for development regions, these devices work excellently for local community networks:

```bash
# Serval DNA daemon configuration example
# Located at /etc/serval/serval.conf

# Configure mesh interface
interface=wlan0
# Enable automatic peer discovery
discovery=yes
# Set mesh network name
meshname=community-mesh-01
# Enable voice routing
voice=enabled
```

The Rhizome protocol ensures data eventually reaches its destination through store-and-forward mechanisms, even when direct paths don't exist.

## Building Custom Mesh Networks with Python

For developers wanting to build custom solutions, the `meshshake` library provides Python tools for experimental mesh networking:

```python
import meshshake

# Initialize mesh node
node = meshshake.Node(
    interface='wlan0',
    protocol='olsr',  # Optimized Link State Routing
    mesh_id='iran-emergency-mesh'
)

# Start mesh advertisement
node.start_advertising()

# Send message to mesh
def send_emergency_message(message, recipients):
    """Broadcast message across mesh network"""
    packet = meshshake.Packet(
        content=message,
        ttl=5,  # hops
        encryption='aes256'
    )
    
    for recipient in recipients:
        node.send(packet, recipient)

# Listen for incoming messages
@node.on_message
def handle_incoming(msg, sender):
    print(f"Received from {sender}: {msg.content}")
```

## Syncthing: Offline File Synchronization

When internet is unavailable but local networks exist, Syncthing provides decentralized file synchronization across devices without cloud dependencies:

```yaml
# Syncthing configuration snippet
# ~/.config/syncthing/config.xml

<device id="DEVICE-ID-1" name="laptop">
    <address>dynamic</address>
    <compress>always</compress>
    <encryption>
        <mode>trusteddevice</mode>
    </encryption>
</device>

<folder id="emergency-docs" path="~/EmergencyDocs">
    <device id="DEVICE-ID-1"></device>
    <device id="DEVICE-ID-2"></device>
    <ignorePerms>false</ignorePerms>
    <ignoreDelete>false</ignoreDelete>
</folder>
```

Key advantages for emergency scenarios:
- No central server required
- Automatic discovery on local networks
- End-to-end encryption by default
- Selective synchronization (only needed folders)
- Works over Wi-Fi Direct on Android

## Offline Maps and Navigation

Internet shutdowns often disrupt navigation and mapping services. Pre-cached offline maps become essential:

### OsmAnd: Offline GPS Navigation

```bash
# OsmAnd can be installed from F-Droid
# Maps are downloaded directly to device storage
# Supports:
# - Turn-by-turn navigation offline
# - Vector maps (searchable)
# - Custom POI markers for meeting points
# - Track recording for creating mesh node maps
```

For community coordination, exporting custom maps with marked meeting points, safe houses, and medical facilities enables navigation without connectivity.

## Implementing Mesh VPN for Network Extension

When some community members maintain internet access (via satellite, international borders, or cached content), mesh VPN solutions can extend connectivity:

```bash
# WireGuard mesh setup example
# Install on each node

# Node A configuration (10.0.0.1)
[Interface]
Address = 10.0.0.1/24
PrivateKey = <node-a-private-key>
ListenPort = 51820

# Node B peer (10.0.0.2)
[Peer]
PublicKey = <node-b-public-key>
AllowedIPs = 10.0.0.2/32, 10.0.0.0/24
Endpoint = 192.168.1.100:51820  # Local IP when mesh connected
PersistentKeepalive = 25

# This allows traffic to flow between mesh nodes
# extending network access to all connected devices
```

## Practical Preparation Checklist

Technical preparation before an emergency involves:

- **Pre-installed applications**: Briar, Serval, Syncthing on all devices
- **Contact exchanges**: QR code exchanges in safe environments
- **Offline content**: Downloaded maps, documents, and guides
- **Local caches**: Browsers with fully cached critical websites
- **Hardware redundancy**: Multiple devices with different capabilities
- **Practice sessions**: Regular mesh network testing within community

## Limitations and Threat Considerations

Mesh networking provides resilience but comes with constraints:

- **Range limitations**: Bluetooth (~10m), Wi-Fi Direct (~200m), dedicated mesh hardware (several km with line-of-sight)
- **Timing dependencies**: Messages propagate only when nodes are in range
- **Device discovery**: Initial contact requires physical proximity
- **Metadata risks**: Even encrypted traffic reveals who communicates with whom

For high-risk scenarios, combining mesh networking with appropriate operational security—limiting communication to essential messages, using dead drops, and rotating devices—reduces exposure.

## Conclusion

Maintaining communication during internet shutdowns requires preparation and understanding of decentralized technologies. Mesh networking tools like Briar and Serval, combined with offline file synchronization via Syncthing, provide developers and power users with robust alternatives when traditional infrastructure fails. The key lies in regular practice, community coordination, and understanding both the capabilities and limitations of these technologies.

Building resilience into communication infrastructure before emergencies occur ensures that communities can maintain coordination when it matters most.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
