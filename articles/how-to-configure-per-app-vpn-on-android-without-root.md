---

layout: default
title: "How to Configure Per App VPN on Android Without Root."
description: "Learn how to set up per-app VPN on Android devices without root access. This guide covers Android's built-in VPN settings, supported apps, and configuration methods for selective VPN routing."
date: 2026-03-18
author: "Privacy Tools Guide"
permalink: /how-to-configure-per-app-vpn-on-android-without-root/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

When you use a VPN on Android, you typically route all your device traffic through the encrypted tunnel. This is great for privacy, but it can slow down streaming apps, cause issues with local network devices, or waste bandwidth on apps that don't need protection. Per-app VPN solves this problem by letting you choose which apps use the VPN and which connect directly to the internet.

The good news is that Android has built-in support for per-app VPN configuration without requiring root access. This guide walks you through setting up selective VPN routing on your Android device in 2026.

## Understanding Per App VPN on Android

Android's per-app VPN feature, introduced in Android 7 (Nougat) and improved in subsequent versions, allows you to create a VPN profile that only certain apps can use. This is different from a traditional VPN that routes everything through the tunnel.

### Why Use Per App VPN?

There are several practical reasons to configure per-app VPN:

**Bandwidth Conservation**: Streaming services like YouTube or Netflix consume massive bandwidth. By excluding them from the VPN, you can save your VPN data allowance and reduce buffering.

**Local Network Access**: Smart home devices, printers, and NAS storage often require direct local network access. A full-device VPN can break this connectivity.

**Performance Optimization**: Some apps don't need encryption and work faster with direct connections. Gaming apps and real-time trading applications often perform better outside VPN tunnels.

**Split Tunneling by App**: You can route banking apps through the VPN for security while allowing other apps to connect directly.

## Prerequisites for Per App VPN

Before configuring per-app VPN, ensure you have:

1. **Android 7.0 or higher** - Per-app VPN requires Android 7.0 (Nougat) or later
2. **A VPN app** - Any VPN provider that supports the Android VPNService API
3. **The apps you want to configure** - Know which apps need VPN and which don't

## Configuring Per App VPN Using Android's Built-in Settings

Here's how to set up per-app VPN on your Android device:

### Step 1: Set Up Your VPN App First

First, install and configure your preferred VPN app from the Google Play Store. Popular options include:

- NordVPN
- ExpressVPN
- ProtonVPN
- WireGuard
- OpenVPN for Android

Configure the VPN app normally and ensure it connects successfully before proceeding.

### Step 2: Access VPN Settings

Navigate to your Android settings:

```bash
Settings → Network & Internet → VPN
```

Or on some devices:

```bash
Settings → Connections → VPN
```

### Step 3: Configure Per-App VPN

For Android 11 and above:

1. Tap the settings gear icon next to your connected VPN
2. Look for "Per-app VPN" or "Split tunneling" option
3. Tap "Add app" to select which apps should use the VPN
4. Select the apps from your installed applications list
5. Confirm your selections

For Android 10 and earlier:

1. Long-press your VPN profile
2. Select "Configure per-app VPN" or "Split tunneling"
3. Choose apps to include or exclude

### Step 4: Test Your Configuration

After configuring:

```bash
# Check which apps are using VPN
# Open your VPN app and verify connected status
# Test with an app inside the VPN tunnel
# Test with an app outside the VPN tunnel
```

## Using VPN Apps with Built-in Per-App Features

Many modern VPN apps include their own per-app configuration interfaces. Here's how to use these features:

### NordVPN Split tunneling

```bash
# NordVPN Android app
Settings → Split tunneling → Add apps
# Select apps to exclude from VPN
```

### WireGuard Configuration

If using WireGuard, you can configure per-app rules in the WireGuard config file:

```ini
[Interface]
PrivateKey = <your-private-key>
Address = 10.0.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = <server-public-key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0

# Exclude specific apps from tunnel
# Note: This requires root on most devices
ExcludeApp = com.netflix.android
ExcludeApp = com.google.android.youtube
```

### OpenVPN for Android Per-App Settings

```bash
# In OpenVPN for Android app
Settings → Per-app settings
# Configure which apps connect through VPN
# Set different rules for each profile
```

## Advanced: Using Sharik for Android Network Sharing

For advanced users who want more control, Sharik is an open-source tool that can help:

```bash
# Install Sharik from F-Droid or compile from source
# Configure network sharing rules
# Set per-app restrictions
```

## Troubleshooting Common Issues

### Apps Not Connecting Through VPN

If your selected apps aren't using the VPN:

1. **Check VPN is connected**: Ensure the VPN shows as connected in the status bar
2. **Verify app selection**: Go back to VPN settings and confirm apps are selected
3. **Restart the VPN**: Disconnect and reconnect the VPN profile
4. **Check app permissions**: Some apps may need VPN permission

### VPN Connection Drops

To maintain stable per-app VPN:

```bash
# Check your VPN app settings
# Enable "Kill switch" or "Block if disconnected"
# Enable "Always-on VPN" in Android settings
Settings → Network & Internet → VPN → Always-on VPN
```

### DNS Leaks in Per-App VPN

Ensure your per-app VPN is properly configured:

```bash
# Test for DNS leaks
# Visit https://dnsleaktest.com
# Verify the DNS server matches your VPN provider
```

## Security Considerations

When using per-app VPN, keep these security points in mind:

1. **Always-on VPN**: Enable this feature in Android settings to prevent accidental data leaks
2. **Kill switch**: Use a VPN app with kill switch to block traffic if VPN disconnects
3. **Split tunneling risks**: Be careful about which apps you exclude—banking apps should typically use VPN
4. **Verify configuration**: Regularly test that excluded apps are actually connecting directly

## Which Apps Should Use VPN?

Here's a recommended split:

| Category | Use VPN | Direct Connection |
|----------|---------|-------------------|
| Banking & Finance | ✅ | |
| Messaging (Signal, WhatsApp) | ✅ | |
| Email | ✅ | |
| Streaming (Netflix, YouTube) | | ✅ |
| Gaming | | ✅ |
| Smart Home | | ✅ |
| Social Media | ✅ | |
| Browsing | ✅ | |

## Conclusion

Per-app VPN on Android without root is a powerful feature that gives you granular control over your network traffic. By following this guide, you can optimize your VPN experience—protecting sensitive apps while maintaining performance for bandwidth-heavy applications.

Remember to test your configuration regularly and adjust which apps use the VPN based on your needs. With proper per-app VPN setup, you get the best of both worlds: security where you need it and performance where you don't.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
