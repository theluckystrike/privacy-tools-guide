---
layout: default
title: "How to Configure Per-App VPN on Android Without Root"
description: "Learn how to set up per-app VPN routing on Android devices without rooting. This guide covers built-in Android features, VPN apps with per-app support, and configuration methods for developers and power users."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-configure-per-app-vpn-on-android-without-root/
categories: [guides, privacy, android]
reviewed: true
intent-checked: true
voice-checked: true
---

{% raw %}

Per-app VPN routing lets you control which applications send traffic through your VPN tunnel and which bypass it. This capability matters for developers testing API integrations, privacy-conscious users isolating sensitive apps, and security professionals segmenting network traffic. Android provides native mechanisms for this, and several VPN implementations support per-app routing without requiring root access.

## Understanding Android's VPN Architecture

Android's `VPNService` API, introduced in Android 4.0 (API level 14), allows applications to create persistent VPN interfaces. Unlike the older `VpnService` APIs that merely routed all system traffic, newer Android versions expanded this to support per-application rules.

The key distinction lies in how you configure routing:

```kotlin
// Basic VPN service setup in Android
val builder = Builder()
    .addAddress("10.0.0.2", 32)
    .addRoute("0.0.0.0", 0)
    .addDnsServer("8.8.8.8")
    .setSession("MyVPN")
    .establish()
```

This creates a system-wide VPN tunnel. To implement per-app routing, you need either an app that supports this natively or Android's built-in per-app VPN settings introduced in Android 5.0 (API level 21).

## Method 1: Using Android's Native Per-App VPN

Android allows you to configure per-app VPN through Settings without installing additional software. This works with any VPN app that uses the standard `VPNService` API.

### Configuration Steps

1. Install your preferred VPN application (WireGuard, OpenVPN Connect, or your VPN provider's app)
2. Connect to the VPN server within the app
3. Navigate to **Settings → Network & Internet → VPN** on Android 12+, or **Settings → Connections → VPN** on older versions
4. Tap the settings icon next to your active VPN profile
5. Look for "Per-app VPN" or "App-based VPN" settings
6. Select the applications you want to route through the VPN

This method works because Android stores per-app VPN configurations in `/data/misc/vpn/` and applies them at the framework level. The VPN app itself doesn't need special modifications—it simply needs to establish a connection, and Android handles the routing based on your selections.

### Limitations

Android's native per-app VPN has constraints:
- Not all VPN apps expose this configuration through their settings
- You cannot selectively route traffic for system apps
- The feature behavior varies across Android versions and manufacturer skins

## Method 2: WireGuard for Per-App Routing

WireGuard offers one of the cleanest implementations for per-app VPN without root. While WireGuard itself is a VPN protocol, its Android client supports per-app routing on devices running Android 10+.

### Installation and Configuration

Install WireGuard from the Google Play Store or F-Droid:

```bash
# If using ADB to install
adb install WireGuard.apk
```

Create a configuration file (`wg0.conf`):

```ini
[Interface]
PrivateKey = <your-private-key>
Address = 10.0.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = <server-public-key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0

# Per-app routing configuration
[Peer]
# Split tunnel for specific apps
PublicKey = <split-tunnel-server-key>
Endpoint = split.vpn.example.com:51820
AllowedIPs = 10.0.0.0/24
```

Import this configuration into the WireGuard app and activate the tunnel. WireGuard's Android implementation allows you to choose between "all traffic" and "split tunnel" modes, though the UI for per-app selection is more limited than dedicated enterprise solutions.

## Method 3: Developer Approach Using VPNService

For developers building custom solutions, the Android `VPNService` API provides full control over traffic routing. Here's a practical example:

```kotlin
import android.net.VpnService

class PerAppVpnService : VpnService() {
    
    private var vpnInterface: ParcelFileDescriptor? = null
    
    override fun onCreate(): Int? {
        return super.onCreate()
    }
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        startVpn()
        return START_STICKY
    }
    
    private fun startVpn() {
        val builder = Builder()
            .setSession("PerAppVPN")
            .addAddress("10.8.0.2", 32)
            .addRoute("0.0.0.0", 0)
            .addDnsServer("1.1.1.1")
            .setMtu(1500)
            .allowFamily(AF_INET)
            .allowFamily(AF_INET6)
        
        // Exclude specific apps from VPN
        // Note: This requires the app to be configured as a "managed" VPN
        // through system settings, not programmatically
        
        vpnInterface = builder.establish()
    }
    
    override fun onDestroy() {
        vpnInterface?.close()
        super.onDestroy()
    }
}
```

To actually implement per-app routing programmatically, you would need to work with Android's `VpnService.Builder` methods that became available in later API levels:

```kotlin
// Android 14+ (API 34) allows more granular control
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
    builder.setMetered(false)
}
```

The key limitation remains: Android doesn't provide a public API for apps to programmatically specify which other apps should use the VPN. This must be done through system settings by the user.

## Method 4: Enterprise VPN Solutions

Several enterprise-focused VPN clients offer robust per-app routing without requiring root:

- **Tailscale**: Built on WireGuard, supports per-device and can route traffic based on ACLs
- **Cloudflare WARP**: Offers zerotrust per-app access in enterprise plans
- **Pulse Secure / Cisco AnyConnect**: Enterprise mobility solutions with app-based routing

For personal use without enterprise licensing, these typically require subscription costs but provide the most reliable per-app functionality.

## Practical Use Cases

### Development Environments

Route only your API testing apps through a development VPN while keeping your banking and communication apps on your regular connection:

```bash
# Example: Route only development traffic
AllowedIPs = 10.0.0.0/16, 192.168.0.0/16
# Excludes 0.0.0.0/0 default route - only routes specified subnets
```

### Privacy Isolation

Route privacy-sensitive applications (browsers, messaging apps) through a VPN while allowing local network access for smart home devices that don't support VPN:

```ini
[Interface]
PrivateKey = <key>
Address = 10.0.0.2/32

[Peer]
PublicKey = <vpn-server-key>
Endpoint = privacy.vpn.example.com:51820
AllowedIPs = 0.0.0.0/0  # Route all traffic
PersistentKeepalive = 25

# Split: Local network excluded
# Configure in app settings to exclude 192.168.x.x ranges
```

## Troubleshooting Common Issues

### Apps Not Respecting VPN Configuration

1. Verify the VPN shows as connected in the system tray
2. Check that apps aren't using hardcoded network configurations
3. Some apps bypass VPN when they detect captive portals
4. Manufacturer battery optimization may kill VPN services—add your VPN app to battery whitelist

### Split Tunneling Not Working

Split tunneling requires explicit support in the VPN client. If your current VPN app lacks this feature:
- Switch to WireGuard which has native split tunnel support
- Use a different VPN provider that exposes per-app settings
- Consider using a proxy app for specific applications instead

### DNS Leaks

Per-app VPN configurations can sometimes leak DNS queries. Ensure your VPN configuration includes DNS servers that override system defaults:

```ini
[Interface]
Address = 10.0.0.2/32
DNS = 1.1.1.1, 8.8.8.8
```

## Security Considerations

Per-app VPN implementations on non-rooted Android devices have inherent limitations:

- System-level traffic (background services, Google Play Services) typically cannot be routed selectively
- Some apps implement certificate pinning which may behave differently when traffic is routed through VPN
- VPN configuration visibility to other apps is limited—verify your VPN is actually active before transmitting sensitive data

For threat models requiring complete network isolation, consider dedicated devices or hardware solutions rather than relying solely on per-app VPN.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
