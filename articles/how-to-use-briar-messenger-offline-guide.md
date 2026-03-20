---

layout: default
title: "How to Use Briar Messenger Offline: A Developer's Guide"
description: "A comprehensive technical guide to using Briar messenger for offline-first, decentralized communication via Bluetooth and Wi-Fi Direct."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-use-briar-messenger-offline-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---


{% raw %}

To use Briar offline, install the app from F-Droid or Google Play, enable Bluetooth or Wi-Fi Direct in Settings, and add contacts by scanning QR codes or exchanging contact links while physically nearby. Briar then forms an encrypted mesh network between devices, delivering messages without any internet connection. For a technical deep dive into Briar's architecture and protocol design, see the [Briar Messenger Offline Mesh Review](/privacy-tools-guide/briar-messenger-offline-mesh-review/).

## Installation and Initial Setup

Briar is available for Android through the F-Droid repository and Google Play Store. For developers who prefer building from source, the project provides instructions for compiling the APK:

```bash
# Clone the Briar repository
git clone https://code.briarproject.org/briar/briar.git
cd briar

# Build the Android APK
./gradlew :briar-android:assembleDebug

# Install on connected device
adb install briar-android/build/outputs/apk/debug/briar-android-debug.apk
```

After installation, creating an identity requires establishing a strong passphrase. This passphrase encrypts your local database, protecting your messages and contacts if your device falls into unauthorized hands. Unlike cloud-based services, Briar stores everything locally by default.

## Adding Contacts Offline

Briar contact addition works entirely offline through QR codes or manual key exchange. When adding a contact, both devices generate temporary Bluetooth or Wi-Fi Direct connections to exchange cryptographic keys.

The process involves these steps:

1. Open Briar on both devices
2. Navigate to Contacts → Add Contact
3. Select the connection method (QR code or link)
4. For QR mode, scan the other device's code
5. For link mode, copy and paste the contact URL

The contact URL contains your public key and connection metadata. You can share this URL through any channel—email, USB drive, or even printed paper. The URL itself reveals no sensitive information; only the holder can complete the pairing process.

## Configuring Offline Transport

Briar's offline capabilities depend on proper configuration of transport protocols. The app supports three transport mechanisms with distinct characteristics.

### Bluetooth Mode

Bluetooth offers the lowest bandwidth but works at greater distances than Wi-Fi Direct, typically up to 100 meters in optimal conditions. Enable Bluetooth transport through Settings → Connectivity → Bluetooth. The app will automatically discover and connect to nearby Briar users when Bluetooth is active.

```bash
# Verify Bluetooth is enabled on your Android device
# Briar automatically manages Bluetooth discovery
# No command-line configuration required
```

Bluetooth works particularly well in scenarios where devices are scattered across a building or outdoor area. The mesh networking capability means messages can hop through multiple devices to reach their destination.

### Wi-Fi Direct Mode

Wi-Fi Direct provides higher bandwidth at shorter ranges, typically 200 meters under ideal conditions. Enable this through Settings → Connectivity → Wi-Fi Direct. Unlike Bluetooth, Wi-Fi Direct allows direct device-to-device connections without the pairing overhead.

Wi-Fi Direct excels for scenarios requiring faster message synchronization, such as sharing media or larger text messages between a small group.

### Internet (Tor) Mode

When internet connectivity exists, Tor-based transport provides anonymous routing while maintaining the same end-to-end encryption. Briar includes a built-in Tor client (Obfs4 proxy), requiring no external configuration.

Configure Tor settings through Settings → Connectivity → Internet. You can specify whether Briar uses internet transport automatically, on-demand, or only when Wi-Fi is available.

## Mesh Networking Behavior

Understanding mesh propagation helps you optimize network topology for your use case. Briar uses a store-and-forward mechanism where each device stores messages until it can forward them to another connected device.

The protocol delivers messages even with intermittent connectivity, discovers routes automatically, deduplicates messages at each hop, and maintains forward secrecy through rotating session keys.

When multiple devices form a mesh, messages propagate automatically. If you move through an area with Briar users, your device will sync with theirs, picking up any messages transmitted while you were out of range.

## Security Tips

For enhanced security when using Briar:
- Use a dedicated device with full-disk encryption enabled
- Set a strong passphrase for Briar's local database
- Be aware that Bluetooth/Wi-Fi radio activity is visible to nearby observers

For a full security model analysis including threat model details, see the [Briar Messenger Offline Mesh Review](/privacy-tools-guide/briar-messenger-offline-mesh-review/).

## Advanced Configuration

### Customizing Sync Behavior

Power users can adjust synchronization parameters through hidden settings. Access developer options by tapping the version number in Settings repeatedly:

```bash
# These settings control mesh sync behavior
# Higher intervals save battery but reduce mesh responsiveness
briar.sync.interval.foreground=30000
briar.sync.interval.background=300000
```

### Network Timeout Configuration

Adjust connection timeouts based on your use case:

```bash
# Shorter timeouts for mobile mesh scenarios
briar.transport.timeout=15000

# Longer timeouts for fixed infrastructure mesh
briar.transport.timeout=60000
```

### Logging and Debugging

Enable debug logging for troubleshooting:

```bash
# Access debug logs through Android's logcat
adb logcat | grep org.briarproject

# Filter specific components
adb logcat | grep -E "(briar|bramble)"
```

## Integration Possibilities

For developers, Briar offers integration opportunities through its API (available in beta). The messaging API allows programmatic message sending and receiving, enabling automated workflows:

```java
// Pseudocode for API usage (beta)
BriarContext context = briar.connect(identity);
MessageEndpoint endpoint = context.getMessageEndpoint();

endpoint.registerMessageType(Message.TYPE_TEXT, 
    (message) -> { processMessage(message); });
```

This enables bots, automated alerts, and integration with existing systems.

## Performance Characteristics

Understanding Briar's performance helps set expectations:

- Message latency: Near-instantaneous on local mesh; minutes to hours for propagation
- Storage: Messages are stored locally; database size grows with message history
- Battery: Bluetooth and Wi-Fi scanning consume power; adjust sync intervals for battery life
- Scalability: Meshes work well with dozens of devices; performance degrades with hundreds

For large groups, periodic synchronization through Wi-Fi Direct provides better performance than maintaining continuous Bluetooth mesh connections.

## Related Reading

- [Bitwarden Vault Export Backup Guide: Complete Technical.](/privacy-tools-guide/bitwarden-vault-export-backup-guide/)
- [VPN Warrant Canary: What It Means and Why It Matters](/privacy-tools-guide/vpn-warrant-canary-what-it-means/)
- [Best VPN for Linux Desktop: A Developer Guide](/privacy-tools-guide/best-vpn-for-linux-desktop/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
