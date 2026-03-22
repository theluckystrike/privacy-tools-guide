---
layout: default
title: "How to Set Up Encrypted DNS-over-HTTPS (DoH) on All Devices"
description: "Step-by-step DoH setup: macOS, Windows, Linux, iOS, Android, routers. Provider comparison (Cloudflare, NextDNS, Quad9). Verification steps included"
date: 2026-03-20
last_modified_at: 2026-03-20
author: theluckystrike
permalink: /how-to-set-up-encrypted-dns-over-https-doh-on-all-devices-guide/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---

{% raw %}

DNS (Domain Name System) queries reveal every website you visit. Without encryption, your ISP, network administrators, and anyone monitoring traffic sees a complete log: "User visited reddit.com at 3:47pm, github.com at 4:02pm, banking.example.com at 4:15pm." DNS-over-HTTPS (DoH) encrypts these queries, protecting your browsing privacy from ISPs and network snoopers.

This guide covers step-by-step setup for all devices—macOS, Windows, Linux, iOS, Android, and routers. DoH is the most impactful privacy improvement you can make with minimal effort. Implementing it across your devices costs nothing and takes 1-2 hours.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Understand DNS Encryption

Standard DNS queries travel unencrypted over UDP port 53. Any device on your network—including your router, ISP infrastructure, or network sniffer—can see every query. Schools, workplaces, and ISPs routinely log DNS queries for monitoring or control purposes.

DNS-over-HTTPS (DoH) wraps DNS queries inside HTTPS encryption. Your DNS query to resolve example.com is now encrypted the same way as web traffic to example.com. ISPs see only that you connected to the DNS provider; they don't see which domains you queried.

The privacy gain is substantial. ISPs can no longer build profiles of your browsing. Advertisers cannot use ISP data to target you. Schools and workplaces cannot secretly monitor your browsing. Governments in countries with internet surveillance have reduced visibility into citizen activity.

Performance impact is negligible. HTTPS adds ~10-50ms to initial queries (amortized across all browsing because results are cached). You won't notice the difference.

## DNS Provider Comparison

### Cloudflare 1.1.1.1 (Fast and Free)

Cloudflare is the largest DoH provider, operated by a company with strong privacy policies.

**Features:**
- Free to all users (no account required)
- Very fast: 99.99% uptime, global anycast network
- Privacy: Cloudflare doesn't log IP addresses (independent audit)
- Malware protection: Blocks known malicious domains
- Adult filter available (optional)
- No account needed; just point to their servers

**Privacy policy:**
- Cloudflare publishes detailed transparency reports
- Committed to not logging your queries
- Third-party audits verify claims
- US jurisdiction (less protective than some alternatives)

**DNS addresses:**
- IPv4: 1.1.1.1, 1.0.0.1
- IPv6: 2606:4700:4700::1111, 2606:4700:4700::1001
- DoH endpoint: https://cloudflare-dns.com/dns-query

**Cost:** Free.

**Best for:** Individuals wanting free, fast DoH without account management.

### NextDNS (Configurable, Detailed Control)

NextDNS provides customizable DNS filtering with detailed analytics about your queries.

**Features:**
- Free tier: 300k DNS queries per month (sufficient for single user)
- Paid tier: Unlimited queries ($5.99/month)
- Detailed analytics: See all queries resolved, breakdown by category
- Custom blocking: Block specific domains or domain categories (malware, ads, gambling, etc.)
- Parental controls: Block adult sites, control screen time
- Logs available for 90 days
- Apps available for all platforms (easier setup)
- DoH and DoT (DNS-over-TLS) support

**Privacy policy:**
- NextDNS can see all queries (by design)
- They don't share data with advertisers
- Queries retained for 90 days minimum
- Less privacy-focused than Cloudflare (more data retention)
- US jurisdiction

**Cost:** Free tier (300k/month), $5.99/month unlimited.

**Setup complexity:** Requires account creation and profile configuration. Slightly more complex than Cloudflare but provides more control.

**Best for:** Users wanting detailed filtering and analytics. Worth the cost for families or organizations.

### Quad9 (Privacy and Security Focused)

Quad9 prioritizes privacy and security, audited by independent researchers.

**Features:**
- Free to all users
- Privacy policy emphasizes minimal data collection
- Malware/botnet protection (automatically blocks malicious domains)
- No data selling to third parties
- Encrypted: DoH and DoT support
- Open-source implementation
- No filtering (unlike NextDNS)
- Headquartered in Switzerland (strong privacy laws)

**Privacy policy:**
- Anonymizes logs (doesn't retain IP addresses)
- Independent privacy audits published
- Non-profit governance model (no profit motive to monetize data)
- Swiss jurisdiction (excellent privacy protections)
- Most privacy-protective option among major providers

**DNS addresses:**
- IPv4: 9.9.9.9, 149.112.112.112
- IPv6: 2620:fe::fe
- DoH endpoint: https://dns.quad9.net/dns-query

**Cost:** Free.

**Best for:** Privacy advocates prioritizing non-profit governance and Swiss jurisdiction over additional features.

### Step 2: macOS Setup

### Method 1: System Settings (iOS 14+ and macOS Monterey+, Easiest)

1. Open System Settings (or System Preferences)
2. Navigate to Network (left sidebar)
3. Select your WiFi network, click "Advanced"
4. Go to DNS tab
5. Click "+" button to add DNS server
6. Enter DoH endpoint: https://cloudflare-dns.com/dns-query (for Cloudflare)
7. Click OK to save

This native method automatically encrypts DNS queries on the device.

**Limitations:** This sets DNS for connected network only. You need to repeat for each network you join.

### Method 2: Encrypted DNS Profile (System-Wide, Advanced)

For encryption on all networks including cellular hotspots:

1. Download Cloudflare WARP app from App Store
2. Install and open the app
3. Enable "1.1.1.1 for Families" option in settings (optional malware blocking)
4. App automatically routes all DNS through encrypted connection

Alternatively, use Quad9 native macOS app or NextDNS app for similar functionality.

### Method 3: Manual Security Configuration (Technical)

1. Open Terminal
2. Create DNS configuration:
```bash
# Enable encrypted DNS for all interfaces
sudo defaults write /Library/Preferences/SystemConfiguration/com.apple.dnssd.plist \
  DNSSDDenyInterfaces -array-add "en0"

# Configure DoH endpoint (requires custom profile, advanced)
```

This method requires creating a configuration profile (XML file) and installing it system-wide. Use GUI methods unless you're comfortable with macOS configuration.

### Step 3: Windows Setup

### Method 1: Network Settings (Windows 11, Easiest)

1. Open Settings (Win+I)
2. Navigate to Network & Internet → WiFi
3. Select connected network → Properties
4. Scroll to "DNS Server Assignment"
5. Click "Edit"
6. Change setting to "Encrypted (DoH)"
7. Enter DoH address: https://cloudflare-dns.com/dns-query
8. Click Save

This enables DoH for current network. Repeat for other networks.

### Method 2: NextDNS or Quad9 App (All Windows Versions)

1. Download NextDNS app or Quad9 app from official website
2. Install and run
3. Select DoH/DoT protocol
4. Configure filtering options (if needed)
5. Enable "Run at startup" for automatic operation

Apps work on all Windows versions and provide system-wide encryption.

### Method 3: PowerShell Configuration (Windows 10/11, Advanced)

1. Open PowerShell as Administrator
2. Enter:
```powershell
# Enable DoH for Cloudflare DNS
Set-DnsClientDohServerAddress -ServerAddress https://cloudflare-dns.com/dns-query -AllowFallbackToUDP $true

# Verify configuration
Get-DnsClientDohServerAddress
```

Repeat for multiple providers or additional DNS servers.

### Step 4: Linux Setup

### Method 1: systemd-resolved Configuration (Modern Linux)

Most modern Linux distributions use systemd-resolved for DNS resolution.

1. Edit `/etc/systemd/resolved.conf` as root:
```bash
sudo nano /etc/systemd/resolved.conf
```

2. Uncomment and modify DNS settings:
```
DNS=1.1.1.1#cloudflare-dns.com 1.0.0.1#cloudflare-dns.com
DNSSEC=yes
DNSSECNegativeTrustAnchors=
FallbackDNS=
```

The `#` followed by domain name specifies the DoH endpoint.

3. Save and restart:
```bash
sudo systemctl restart systemd-resolved
```

4. Verify:
```bash
systemctl status systemd-resolved
resolvectl status
```

### Method 2: stubby (Dedicated DoH Resolver, Recommended)

Stubby is a dedicated DNS resolver supporting DoH:

1. Install stubby:
```bash
# Ubuntu/Debian
sudo apt install stubby

# Fedora/RHEL
sudo dnf install stubby
```

2. Edit `/etc/stubby/stubby.yml`:
```yaml
# Cloudflare configuration
upstream_recursive_servers:
  - address_data: 1.1.1.1
    tls_auth_name: cloudflare-dns.com
    tls_pubkey_pinset:
      - digest: "sha256"
        value: "rBtSQ3e90U+mzJYUJkMKn75edfMUFpwHFFUagKHoVeE="

  - address_data: 1.0.0.1
    tls_auth_name: cloudflare-dns.com
    tls_pubkey_pinset:
      - digest: "sha256"
        value: "rBtSQ3e90U+mzJYUJkMKn75edfMUFpwHFFUagKHoVeE="

# Listen on localhost
listen_addresses:
  - 127.0.0.1@53
  - 0::1@53
```

3. Start and enable:
```bash
sudo systemctl enable stubby
sudo systemctl start stubby
```

4. Configure systemd-resolved to use stubby:
```bash
sudo nano /etc/systemd/resolved.conf
# Add:
DNS=127.0.0.1
FallbackDNS=
DNSSEC=yes
```

5. Restart:
```bash
sudo systemctl restart systemd-resolved
```

### Method 3: NextDNS App (Simple, All Linux)

1. Install NextDNS app from GitHub releases
2. Run and authenticate with account
3. App handles all configuration automatically

### Step 5: iOS Setup

### Method 1: Settings (iOS 14+, System-Wide)

1. Open Settings → VPN & Device Management
2. Tap "DNS Settings" (if visible—available on iOS 15+)
3. Add DNS configuration
4. Select "Encrypted (DoH)"
5. Enter provider details

Not all versions support this directly; iOS 15+ is most reliable.

### Method 2: VPN Configuration Profile (All iOS Versions)

1. Download configuration profile from provider website (Cloudflare, NextDNS)
2. Open the profile in Safari
3. Go to Settings → VPN & Device Management
4. Select the downloaded profile
5. Tap Install
6. Confirm with Face ID/Touch ID

This installs provider-specific encryption settings.

### Method 3: Official Provider App

**Cloudflare WARP:**
1. Download from App Store
2. Enable in app settings
3. Runs automatically on all networks

**NextDNS app:**
1. Download from App Store
2. Create or log into account
3. Select profile and enable
4. App encrypts all DNS

**Quad9:**
1. Download Quad9 app from App Store
2. Enable in settings
3. Automatic encryption on all networks

### Step 6: Android Setup

### Method 1: System Settings (Android 9+)

1. Open Settings → Network & Internet → Private DNS
2. Select "Private DNS provider hostname"
3. Enter DoH provider (dns.cloudflare.com for Cloudflare)
4. Confirm

This applies to all networks automatically.

### Method 2: VPN Configuration

Some providers offer VPN apps that encrypt DNS as part of VPN connection. Use caution—VPN routes all traffic through provider servers, not just DNS.

For DNS-only encryption, use Method 1 or official app.

### Method 3: NextDNS App

1. Download from Play Store
2. Create account or use existing
3. Enable "Connect" in app
4. App encrypts all DNS queries

### Step 7: Router Configuration (Network-Wide DoH)

Setting DoH on your router encrypts DNS for all connected devices automatically.

### Supported Routers

- Asus (many models support DoH)
- Ubiquiti EdgeRouter
- OpenWrt (community firmware)
- Firewalla
- Synology NAS (can serve as router)

### Example: Asus Router

1. Connect to router admin panel (192.168.1.1 or similar)
2. Navigate to WAN settings
3. Find DNS settings
4. Select "Manual DNS Configuration"
5. Enter primary and secondary DoH providers:
 - Cloudflare: dns.cloudflare.com
 - Quad9: dns.quad9.net
6. Save and reboot router

All connected devices now use encrypted DNS automatically.

### Example: OpenWrt (Advanced)

For custom firmware like OpenWrt:

1. SSH into router:
```bash
ssh root@192.168.1.1
```

2. Install dnscrypt-proxy:
```bash
opkg install dnscrypt-proxy
```

3. Edit `/etc/config/dnscrypt-proxy`:
```
config dnscrypt_proxy
    option enabled 1
    option server 'cloudflare'
    option listen_addr '127.0.0.1:53'
```

4. Restart service:
```bash
/etc/init.d/dnscrypt-proxy restart
```

### Step 8: Verification: Confirm DoH is Working

### Test 1: DNS Query Encryption Check

Use an online DoH verification tool:
1. Visit: https://doh.test/
2. Tool shows whether queries are encrypted
3. Confirms DoH provider in use

### Test 2: Command Line (Linux/macOS)

```bash
# Test DNS resolution
nslookup example.com

# Test with specific DoH server (requires dig with DoH support)
dig @1.1.1.1 example.com
```

### Test 3: Network Inspection (Technical)

Use Wireshark packet capture to verify traffic:
1. Capture traffic on your device
2. Filter for DNS traffic (port 53)
3. No plaintext DNS queries should appear
4. HTTPS traffic to DoH provider's IP should be visible

If DNS queries appear in plaintext, DoH isn't enabled or fallback occurred.

## Troubleshooting Common Issues

**DNS not resolving:** Some networks block custom DNS. Try:
- Switching to different DoH provider
- Using network admin credentials if available
- Testing on different network to isolate issue

**Slow internet:** DoH adds minimal latency. If noticeable:
- Confirm DoH endpoint is responsive (ping the IP)
- Switch to geographically closer provider
- Disable malware filtering if enabled (reduces latency)

**App doesn't recognize DoH setting:** Some apps use hardcoded DNS servers:
- Check app privacy settings for DNS options
- Some browsers (Firefox) have DoH settings independent of system
- VPN apps often override system DoH

**Mobile hotspot DNS leaks:** Phone-provided hotspot may override DoH:
- Enable DoH on both phone and connected device
- Use dedicated mobile router with DoH support instead

### Step 9: Privacy Considerations

**DoH doesn't encrypt destination IP:** ISPs still see which servers you connect to (by IP address). DoH protects domain names only, not destination addresses. For complete privacy, combine DoH with VPN.

**VPN adds DoH redundancy:** A VPN encrypts all traffic including DNS, so separate DoH isn't needed. However, DoH provides defense-in-depth: even if VPN fails, DNS remains encrypted.

**Provider trust:** You're trusting the DoH provider with your DNS queries. Choose providers with strong privacy policies:
- Cloudflare and Quad9: Strong independent audits
- NextDNS: Good privacy, but retains logs
- Avoid suspicious providers claiming privacy without audits

### Step 10: Implementation Timeline

**Day 1:** Set up DoH on primary device (phone or laptop) using official app or system settings.

**Day 2-3:** Configure DoH on remaining personal devices (secondary phone, tablet, desktop).

**Day 4-5:** If you have network access, configure router for network-wide DoH.

**Ongoing:** Verify DoH is working monthly using verification tools. Update settings if provider changes recommendations.

## Frequently Asked Questions

**How long does it take to set up encrypted dns-over-https (doh) on all devices?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [How to Set Up Encrypted DNS on All Devices 2026](/privacy-tools-guide/how-to-set-up-encrypted-dns-on-all-devices-2026/)
- [How to Configure DNS Over HTTPS (DoH) for Privacy in 2026](/privacy-tools-guide/how-to-configure-dns-over-https-for-privacy-2026/---)
- [Encrypted DNS over HTTPS on Linux](/privacy-tools-guide/encrypted-dns-over-https-linux-setup)
- [DNS over TLS Setup on Linux](/privacy-tools-guide/dns-over-tls-setup-linux-android/)
- [Encrypted Dns Messaging Combination How To Layer Privacy Pro](/privacy-tools-guide/encrypted-dns-messaging-combination-how-to-layer-privacy-pro/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
```
{% endraw %}
