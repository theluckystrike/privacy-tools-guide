---
layout: default
title: "Privacy Setup for Someone Hiding from Abusive Ex"
description: "A technical guide to hardening your digital presence when escaping domestic violence. Covers device hardening, account isolation, network security, and"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /privacy-setup-for-someone-hiding-from-abusive-ex-comprehensi/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, privacy]
---

{% raw %}

This guide assumes you have access to a trusted computer and can install software. If you are in immediate danger, skip to the emergency section at the end. The technical measures here complement—not replace—professional support and safety planning.
Immediately enable full-disk encryption (FileVault on macOS, BitLocker on Windows, LUKS on Linux), create a new anonymous email account from a safe device, use a password manager to generate strong unique passwords for all accounts, remove all location-sharing features and check for hidden tracking apps, and use Tor for any sensitive communications. Change all recovery methods and security questions, remove the abuser's access to shared accounts, and establish emergency contacts outside the abuser's network. This guide provides immediate and long-term actionable technical steps with code examples for securing your digital safety when leaving an abusive relationship.

## Threat Model

When fleeing an abusive ex who possesses technical knowledge, your threat model includes someone who may have installed spyware, has credentials to shared accounts, understands your digital habits, and could use social engineering against your contacts. Every recommendation below addresses these specific risks.

## Device Hardening

### Clean Device Installation

Never continue using a device your abuser had physical access to. Purchase a new device with cash if possible. Boot into recovery mode and wipe the existing OS completely:

```bash
# On macOS: Hold Cmd+R during boot, select Disk Utility, erase the drive
# On Linux: Use a live USB to run shred or badblocks on the entire drive
sudo shred -v -n 3 /dev/sda
```

For mobile, perform a factory reset after enabling full disk encryption. On Android, verify the bootloader is locked afterward:

```bash
fastboot flashing get_unlock_ability
```

If unlock_ability returns 0, the bootloader remains locked—a positive security indicator.

### Operating System Choice

GrapheneOS on a Pixel device provides strong sandboxing and sensor toggles that physically disable the microphone and cameras. This matters because stalkerware often exploits camera and microphone access. Install GrapheneOS via the official web-based installer:

1. Enable developer settings on the Pixel
2. Visit grapheneos.org/install
3. Follow the web installer instructions (no flashing required)

For desktop, Tails or Qubes OS compartmentalize your activities. Tails wipes memory on shutdown—essential for preventing forensic recovery of your location data.

## Account Isolation

### Email Separation

Create a completely new email address from a device your abuser never touched. Use a privacy-respecting provider like Proton Mail. Enable two-factor authentication with a hardware key, not SMS:

```bash
# Generate a YubiKey-compatible GPG key for authentication backup
gpg --full-generate-key
# Use this key as your 2FA backup, store the revocation certificate offline
```

Never link this new email to your previous accounts. Do not import contacts or emails from your old accounts.

### Phone Number Strategy

Get a new SIM card from a provider that doesn't require ID registration where you live. Use a VoIP number (like MySudo or Google Voice) for all dating and social accounts—keep your real number completely separate. On Android, configure calls to route through the VoIP app only:

```kotlin
// Example: Using Tasker to route calls from unknown numbers to VoIP
Profile: Unknown Caller
Event: Phone Ringing
    IF %CNUMBER !IN %MyContacts
Task: Accept Call
    App Launch: VoIP App
```

This prevents your ex from discovering your new number through social engineering telecom employees.

### Password Manager Segregation

Your old password manager is compromised. Create a fresh vault in Bitwarden or 1Password with a completely new master password. Generate unique 20+ character passwords for every account:

```bash
# Using Bitwarden CLI to generate passwords
bw generate --length 24 --complexity --includeNumber --includeSymbol --includeUppercase --includeLowercase
```

Enable the emergency access feature with a trusted person who understands the sensitivity—but give them instructions to delete the request if they receive it unexpectedly. This serves as a dead man's switch for your digital life.

## Network Security

### VPN Configuration

Route all traffic through a VPN you control—a self-hosted WireGuard server on a VPS purchased with cryptocurrency. This prevents your ISP and anyone monitoring network traffic from seeing your browsing activity:

```ini
# /etc/wireguard/wg0.conf on your VPS
[Interface]
PrivateKey = <your-server-private-key>
Address = 10.0.0.1/24
PostUp = iptables -A FORWARD -i %i -j ACCEPT
PostUp = iptables -A FORWARD -o %i -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <your-client-public-key>
AllowedIPs = 10.0.0.2/32
PersistentKeepalive = 25
```

Configure your client to route all traffic through the tunnel. Set the kill switch to block all non-VPN traffic—this prevents accidental leaks if the connection drops.

### DNS Configuration

Use NextDNS or your own Pi-hole to block tracking domains. Configure it to log nothing:

```yaml
# nextdns.yml configuration
listen: :53
discovery: false
reporting:
  enabled: false
blocks:
  - 1d8c6f16-6466-11ea-8c41-063af41d5f76
```

Blocklists should include known stalkerware domains, advertising trackers, and social media trackers.

### Browser Hardening

Use Firefox with the following about:config changes:

```
privacy.resistFingerprinting = true
network.cookie.cookieBehavior = 1
privacy.trackingprotection.enabled = true
webgl.disabled = true
```

Install uBlock Origin and Privacy Badger. Create a separate browser profile for sensitive activities—bookmark the new email, the VPN status page, and your safety plan.

## Operational Security

### Metadata Stripping

Before sharing any photos, strip EXIF data:

```bash
# Using exiftool
exiftool -all= -overwrite_original image.jpg

# Batch processing
for f in *.jpg; do exiftool -all= -overwrite_original "$f"; done
```

On mobile, use Metapho orexif to remove location data before sending photos to anyone.

### Signal Configuration

Configure Signal with these settings:

1. Enable registration lock with a strong PIN
2. Set disappearing messages to 24 hours for all chats
3. Disable link previews (prevents URL leakage)
4. Use the screen lock feature

Never link Signal to your old phone number.

### Social Media Audit

Search for yourself on social media using your new email and phone number. Set Google Alerts for your name variations. Request account deletion from any platform where your ex might have created lookalike accounts:

```bash
# Use the email you just created to check data broker exposure
curl -X DELETE https://api.example-data-broker.com/v1/profile \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email": "your-new-email@example.com"}'
```

## Emergency Protocols

If you suspect immediate danger:

1. Power off all devices and remove batteries if possible
2. Go to a location your ex doesn't know about
3. Use a library computer or borrowed device to change critical passwords
4. Contact a domestic violence hotline—they have secure communication channels

Save these resources offline:

- National Domestic Violence Hotline: 1-800-799-7233
- Cyber Civil Rights Initiative: 1-844-878-2274


## Related Articles

- [Privacy Setup For Someone Leaving Abusive Relationship Digit](/privacy-tools-guide/privacy-setup-for-someone-leaving-abusive-relationship-digit/)
- [Veterinarian Client Pet Data Privacy Protection Setup Guide](/privacy-tools-guide/veterinarian-client-pet-data-privacy-protection-setup-guide/)
- [How to Use Tails OS for Maximum Privacy Complete Setup Guide](/privacy-tools-guide/how-to-use-tails-os-for-maximum-privacy-complete-setup-guide/)
- [Insurance Agent Client Health Data Privacy Protection Setup](/privacy-tools-guide/insurance-agent-client-health-data-privacy-protection-setup/)
- [Linux Mint Privacy Setup Guide for Beginners](/privacy-tools-guide/linux-mint-privacy-setup-guide-beginners/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
