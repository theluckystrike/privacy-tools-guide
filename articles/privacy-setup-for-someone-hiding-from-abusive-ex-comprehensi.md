---
layout: default
title: "Privacy Setup for Someone Hiding From Abusive Ex."
description: "A practical technical guide for securing devices, accounts, and communications when fleeing an abusive situation. Includes code examples and."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /privacy-setup-for-someone-hiding-from-abusive-ex-comprehensi/
categories: [guides]
reviewed: true
score: 8
---

{% raw %}

When your safety depends on digital privacy, standard security advice falls short. This guide provides actionable technical steps for securing your digital life when leaving an abusive relationship. The focus is on practical implementation—things you can do right now—with code examples tailored for developers and power users.

## Immediate Device Hardening

Your phone and computer are the primary vectors an abuser uses to track you. Start with physical security, then work outward.

### Enable Full Disk Encryption

On macOS, FileVault encrypts your entire drive. Enable it immediately:

```bash
sudo fdesetup enable
```

On Linux with LUKS:

```bash
sudo cryptsetup luksFormat /dev/sdaX
```

Windows users should enable BitLocker via `manage-bde -on C:` in an elevated command prompt.

### Boot Security

Set a firmware password to prevent booting from external media. On Mac, restart and hold Command+R to enter Recovery Mode, then select Firmware Password Utility. For Linux systems, enable Secure Boot and ensure your bootloader is protected with a password.

### Secondary Phone Strategy

Acquire a burner phone for sensitive communications. Use a prepaid SIM card purchased with cash—register it with a pseudonym if your jurisdiction allows. Keep your primary phone powered off when not in use; remove the battery if possible to prevent location tracking via cell tower pings.

## Account Security Overhaul

Abusive ex-partners often have credentials from the relationship period. Assume all shared accounts are compromised until proven otherwise.

### Password Rotation

Generate new passwords for every account using a password manager. Create a unique, high-entropy master password:

```bash
openssl rand -base64 32
```

Enable two-factor authentication on every service that supports it. Prefer authenticator apps over SMS—a SIM swap attack can compromise phone-based 2FA.

### Email Account Isolation

Create a completely new email address from a privacy-respecting provider. ProtonMail or Tutanota offer end-to-end encryption by default. Set up account recovery options using only this new email—never link it to your old accounts.

For added security, access the new email exclusively through Tor Browser:

```bash
# Download Tor Browser from torproject.org
./start-tor-browser --verbose
```

### Account Discovery and Cleanup

Search your existing accounts for shared devices, authorized applications, and recovery options. Revoke access to all unrecognized devices:

- Google: myaccount.google.com/permissions
- Apple: appleid.apple.com
- Microsoft: account.microsoft.com/devices

## Communication Hardening

Your messaging apps carry significant risk. An abuser who previously had access to your phone may have installed spyware or configured message forwarding.

### Signal Configuration

Signal provides end-to-end encryption by default, but configure these settings:

1. Enable "Disappearing Messages" for all conversations
2. Set a registration lock PIN
3. Disable cloud backups in Settings > Privacy

```bash
# Verify Signal installation integrity on Android
adb shell dumpsys package org.thoughtcrime.securesms | grep -i signature
```

### Phone Number Protection

Contact your mobile carrier to add a PIN to your account—this prevents SIM swapping. Request that your number be removed from online directories. Consider moving to a carrier that supports eSIM, allowing you to switch numbers instantly if needed.

## Location Privacy

Modern devices constantly broadcast location data. Disable this systematically.

### Device-Level Location Controls

- **iOS**: Settings > Privacy > Location Services > Disable
- **Android**: Settings > Location > Disable
- **macOS**: System Preferences > Security & Privacy > Privacy > Location Services

Review which apps have location access and revoke it for everything except mapping applications you actively use.

### Metadata Stripping

Photos contain GPS coordinates, device information, and timestamps. Remove metadata before sharing:

```bash
# Using exiftool (brew install exiftool)
exiftool -all= -overwrite_original image.jpg
```

For batch processing:

```bash
find ./photos -type f -exec exiftool -all= -overwrite_original {} \;
```

### WiFi and Bluetooth

Disable WiFi scanning and auto-connect. Avoid public WiFi networks—use a VPN instead. Turn off Bluetooth when not in active use; Bluetooth beacons can track device movement.

## Digital Footprint Reduction

Search for yourself online and request removal of personal information from data broker sites. Use services like JustOptOut or DeleteMe, or submit removal requests directly:

- Acxiom: acxiom.com/opt-out-request
- Experian: experian.com/optout
- Spokeo: spokeo.com/optout

Create new social media accounts with privacy-first settings—private profiles, no location tagging, no mutual connections with old accounts.

## Network Security

Your home network is your castle. Secure it properly.

### Router Configuration

Change the default administrator credentials on your router. Use WPA3 encryption if available, WPA2-AES as a minimum. Create a guest network for visitors:

```bash
# Example router configuration via SSH (varies by manufacturer)
ssh admin@192.168.1.1
> set wireless security wpa3-personal
> set wireless guest-network enable
> set wireless guest-network isolation enable
```

### VPN Implementation

Route all traffic through a reputable VPN. Self-host a WireGuard server for maximum control:

```bash
# Server setup (Ubuntu)
apt install wireguard
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey

# /etc/wireguard/wg0.conf
[Interface]
PrivateKey = <your-private-key>
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client-public-key>
AllowedIPs = 10.0.0.2/32
```

Configure clients to connect automatically on untrusted networks.

## Emergency Response Plan

Prepare for the worst-case scenario:

1. **Cloud Account Backup**: Ensure critical data exists in encrypted cloud storage you alone control
2. **Device Wipe Capability**: Enable Find My Device (iOS) or Find My Device (Android) for remote wipe
3. **Evidence Preservation**: Screenshot any threatening messages, save emails, document harassment before blocking
4. **Safe Computer**: Have access to a trusted friend's computer or library system for sensitive communications

## Conclusion

Digital security for someone fleeing an abusive situation requires paranoia as a feature, not a bug. Every setting, every app, every account needs scrutiny. The steps above provide a strong foundation—but security is ongoing. Review your settings monthly, stay aware of new threats, and prioritize physical safety alongside digital hardening.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
