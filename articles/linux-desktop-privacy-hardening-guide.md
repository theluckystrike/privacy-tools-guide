---
layout: default
title: "Linux Desktop Privacy Hardening Guide"
description: "Ubuntu and Fedora privacy hardening: AppArmor, firewall, disk encryption, telemetry removal, browser sandboxing. CLI examples and step-by-step"
date: 2026-03-20
last_modified_at: 2026-03-20
author: "Privacy Tools Guide"
permalink: /linux-desktop-privacy-hardening-guide/
categories: [guides]
tags: [privacy-tools-guide, tools, best-of, privacy]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---

{% raw %}

Linux desktops run on trust—you control the OS, applications, and permissions. But default installations leak data: Ubuntu's Amazon search suggestions, Fedora's automatic analytics, Firefox's default telemetry, and browser fingerprinting. This guide hardens a Linux desktop against data leaks with practical CLI commands and configuration changes. Expect 2-3 hours for a complete hardening. The result: a system that collects no data about you and refuses unauthorized network connections.


## The Privacy Baseline

Before hardening, understand what your system currently shares:

```bash
# Check running services exposing network ports
sudo netstat -tlnp | grep LISTEN

# Check scheduled tasks running automatically
systemctl --user list-timers
sudo systemctl list-timers

# Check installed telemetry packages (Ubuntu)
apt list --installed | grep -E 'ubuntu|analytics|metric|telemetry'

# Check installed telemetry packages (Fedora)
dnf list installed | grep -E 'abrt|rhel|analytics'
```

Common culprits:

- **Ubuntu**: `popularity-contest` (anonymously reports installed packages), `ubuntu-report` (submits hardware info), `apport` (crash reporting)
- **Fedora**: `abrt` (automatic bug reporting), `dnf-plugins-core` (telemetry in some versions)
- **All distros**: Firefox default telemetry, update checkers, automatic error reports


## Step 1: Disable Ubuntu Telemetry (Ubuntu Only)

Remove packages sending data to Ubuntu/Canonical:

```bash
# Remove data collection packages
sudo apt purge ubuntu-report popularity-contest apport apport-gtk -y

# Disable Ubuntu Pro telemetry
sudo ubuntu-pro disable-esm
sudo ubuntu-pro disable-fips

# Remove Ubuntu Pro service
sudo apt purge ubuntu-advantage-tools -y

# Prevent reinstallation
echo "ubuntu-report" | sudo tee -a /etc/apt/preferences.d/no-telemetry
echo "Package: ubuntu-report" | sudo tee /etc/apt/preferences.d/no-telemetry
echo "Pin: release *" | sudo tee -a /etc/apt/preferences.d/no-telemetry
echo "Pin-Priority: -1" | sudo tee -a /etc/apt/preferences.d/no-telemetry

# Disable MOTD telemetry
sudo chmod -x /etc/update-motd.d/*

# Verify no telemetry services remain
sudo systemctl status ubuntu-report 2>&1 | grep -i inactive
```

**Result:** Ubuntu no longer sends package data or hardware surveys to Canonical.


## Step 2: Disable Fedora Telemetry (Fedora Only)

```bash
# Remove automatic bug reporting daemon
sudo dnf remove abrt abrt-addon-python* -y

# Disable telemetry in DNF
sudo sed -i 's/countme=1/countme=0/' /etc/dnf/dnf.conf

# Remove any preloaded telemetry packages
sudo dnf remove dnf-plugins-extras-repoclosure -y

# Disable ABRT service (if not removed)
sudo systemctl disable abrtd
sudo systemctl stop abrtd

# Verify
grep "countme" /etc/dnf/dnf.conf  # Should show countme=0
```

**Result:** Fedora stops sending package statistics and debug reports.


## Step 3: Harden the Firewall

Linux desktops rarely need inbound ports. A strict firewall rejects unsolicited connections:

**UFW (Ubuntu, simple):**

```bash
# Install UFW
sudo apt install ufw -y

# Enable firewall (outbound allow by default, inbound deny)
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable

# Allow SSH if needed (adjust port if not 22)
sudo ufw allow 22/tcp

# Verify
sudo ufw status

# Example output:
# Status: active
#
# To                         Action      From
# --                         ------      ----
# 22/tcp                     ALLOW       Anywhere
```

**firewalld (Fedora, complex but powerful):**

```bash
# Install and enable firewalld
sudo dnf install firewalld -y
sudo systemctl enable firewalld
sudo systemctl start firewalld

# Set default zone to drop (reject all by default)
sudo firewall-cmd --set-default-zone=drop

# Allow SSH
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=dhcpv6-client  # For IPv6 DHCP

# Verify
sudo firewall-cmd --list-all

# Make changes permanent
sudo firewall-cmd --reload
```

**Result:** Your desktop rejects unsolicited network connections. Only services you explicitly allow through.

**Verify the firewall works:**

```bash
# From another machine, try to connect to random ports
nmap -p 1-10000 your-desktop-ip

# Expected: All ports filtered or closed (good)
```


## Step 4: Enable Full Disk Encryption

Unencrypted disks leak data even when powered off. Full disk encryption (LUKS) encrypts everything except the boot partition.

**For existing systems (after installation):**

```bash
# Check if encryption exists
sudo cryptsetup status /dev/mapper/*

# If not encrypted, encryption requires resizing partitions (complex)
# Easier to reinstall with encryption

# To verify encryption during reinstallation:
# 1. Run Ubuntu/Fedora installer
# 2. At "Installation type", select "Something else"
# 3. Select your main partition, click Edit
# 4. Check "Use as: physical volume for encryption" (LVM)
# 5. Complete installation
```

**For new installations, verify encryption:**

```bash
# List encrypted volumes
sudo lsblk | grep -E "crypt|CRYPT"

# Check encryption status
sudo dmsetup status

# Get encryption algorithm
sudo cryptsetup status /dev/mapper/your-volume
# Should show: cipher=aes-xts-plain64, key size=512 bits (AES-256)

# Verify encrypted home
df -h | grep -i encrypt
```

**Enable swap encryption (if not done automatically):**

```bash
# Disable swap
sudo swapoff -a

# Encrypt swap (only if not already encrypted)
sudo cryptsetup create swapvol /dev/sdX  # Replace X with swap partition
sudo mkswap /dev/mapper/swapvol
sudo swapon /dev/mapper/swapvol

# Make permanent by updating /etc/crypttab and /etc/fstab
sudo nano /etc/crypttab
# Add: swapvol /dev/sdX /dev/urandom swap
```

**Result:** Disk is encrypted with AES-256. Removing the drive and powering it on elsewhere reveals no data.


## Step 5: Enable AppArmor (Ubuntu) or SELinux (Fedora)

Mandatory Access Control (MAC) limits what each application can access. Even if an application is compromised, it can't access the rest of your system.

**AppArmor (Ubuntu):**

```bash
# Install AppArmor
sudo apt install apparmor apparmor-utils apparmor-profiles-extra -y

# Enable AppArmor
sudo systemctl enable apparmor
sudo systemctl start apparmor

# Load strict profiles for common apps
sudo aa-enforce /etc/apparmor.d/usr.bin.firefox
sudo aa-enforce /etc/apparmor.d/usr.bin.chromium
sudo aa-enforce /etc/apparmor.d/usr.bin.man

# View AppArmor status
sudo aa-status

# Example output shows enforced and complain profiles
```

**SELinux (Fedora):**

```bash
# Check SELinux status
getenforce  # Should return "Enforcing"

# If not enforcing:
sudo semanage permissive -a firefox_t  # Or other apps

# View enforcing policies
getsebool -a | grep -i httpd

# Check denial logs
sudo journalctl -u setroubleshoot --no-pager | tail -20
```

**Result:** Applications run in sandboxes. Firefox can't access your SSH keys, documents, or home directory outside its profile.


## Step 6: Harden Firefox

Firefox is your primary internet gateway. Default settings collect telemetry and expose fingerprinting vectors.

```bash
# Open Firefox, go to about:config
# Set these preferences:

# Disable all telemetry
privacy.trackingprotection.enabled = true
datareporting.healthreport.uploadEnabled = false
datareporting.policy.dataSubmissionEnabled = false
toolkit.telemetry.enabled = false
toolkit.telemetry.archive.enabled = false
extensions.activeTheme = firefox-compact-dark  # Minimal fingerprint

# Disable pocket recommendations
extensions.pocket.enabled = false

# Disable search suggestions
browser.search.suggest.enabled = false
browser.search.suggest.enabled.private = false

# Disable DNS-over-HTTPS leaking (use system DNS)
network.trr.mode = 5  # 5 = off, 2 = second choice, 3 = only HTTPs

# Disable WebRTC IP leak
media.peerconnection.enabled = false
# Or if you need WebRTC:
media.peerconnection.ice.default_address_only = true

# Disable fingerprinting scripts
privacy.resistFingerprinting = true

# Disable Canvas fingerprinting
canvas.poisondata = true
```

**Or use a privacy-focused config file:**

```bash
# Create user.js
cat > ~/.mozilla/firefox/PROFILE_NAME/user.js << 'EOF'
// Telemetry disabled
user_pref("datareporting.healthreport.uploadEnabled", false);
user_pref("datareporting.policy.dataSubmissionEnabled", false);
user_pref("toolkit.telemetry.enabled", false);
user_pref("toolkit.telemetry.archive.enabled", false);

// Tracking protection
user_pref("privacy.trackingprotection.enabled", true);

// Pocket disabled
user_pref("extensions.pocket.enabled", false);

// Fingerprinting resistance
user_pref("privacy.resistFingerprinting", true);

// WebRTC
user_pref("media.peerconnection.enabled", false);
EOF

# Restart Firefox to apply
```

**Verify fingerprint resistance:**

```bash
# Visit https://coveryourtracks.eff.org/
# Ideal result: "Untrackable" (difficult to fingerprint)
```


## Step 7: Disable Unnecessary Services

Disable services not needed on a desktop:

```bash
# List running services
systemctl list-units --type=service --state=running

# Disable unnecessary ones:
sudo systemctl disable bluetooth  # Unless you use wireless peripherals
sudo systemctl disable cups  # Unless you print
sudo systemctl disable avahi-daemon  # Unless you use network discovery
sudo systemctl disable isc-dhcp-server  # Unless you run DHCP

# For non-root user, disable user services:
systemctl --user disable evolution-data-server  # Unless you use Evolution email
systemctl --user disable tracker  # File indexer (privacy concern)
systemctl --user disable geoclue  # Location services
```

**Verify:**

```bash
sudo systemctl status bluetooth  # Should show "disabled"
```


## Step 8: Configure Automatic Security Updates

Unpatched systems are vulnerable. Enable automatic security updates:

**Ubuntu:**

```bash
# Install unattended-upgrades
sudo apt install unattended-upgrades -y

# Enable automatic updates
sudo dpkg-reconfigure -plow unattended-upgrades

# Verify it's running
sudo systemctl status unattended-upgrades

# Check logs
sudo tail -20 /var/log/unattended-upgrades/unattended-upgrades.log
```

**Fedora:**

```bash
# Enable automatic updates via DNF
sudo dnf install dnf-automatic -y
sudo systemctl enable dnf-automatic.timer
sudo systemctl start dnf-automatic.timer

# Verify
sudo systemctl status dnf-automatic.timer
```


## Step 9: Enable SSH Hardening (If Needed)

If you use SSH, harden it:

```bash
# Disable password authentication
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# Disable root login
sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin no/' /etc/ssh/sshd_config

# Use only SSH protocol 2
sudo sed -i 's/#Protocol 2/Protocol 2/' /etc/ssh/sshd_config

# Disable X11 forwarding (unless needed)
sudo sed -i 's/X11Forwarding yes/X11Forwarding no/' /etc/ssh/sshd_config

# Restart SSH
sudo systemctl restart ssh  # Ubuntu
sudo systemctl restart sshd  # Fedora

# Verify configuration
sudo sshd -t  # Should output nothing if valid
```


## Step 10: Lock Down Permissions

Restrict file permissions to prevent unauthorized access:

```bash
# Home directory (user only)
chmod 700 ~

# SSH configuration (user only)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_rsa

# Private files
chmod 600 ~/.gnupg/*

# Check for world-readable sensitive files
find ~ -type f -perm /077 -print | grep -E "ssh|gpg|key|token|password|secret"

# Remove world-readable permissions
find ~ -type f -perm /077 -exec chmod go-rwx {} \;
```


## Step 11: Enable Mandatory Security Monitoring

Track security events in a central log:

```bash
# Install auditd
sudo apt install auditd  # Ubuntu
sudo dnf install audit  # Fedora

# Enable auditing
sudo systemctl enable auditd
sudo systemctl start auditd

# Monitor file access
sudo auditctl -w /etc/shadow -p wa -k shadow_access
sudo auditctl -w /etc/passwd -p wa -k passwd_access

# View audit logs
sudo ausearch -k shadow_access  # Shows attempts to modify passwords

# Make rules permanent
sudo nano /etc/audit/rules.d/hardening.rules
# Add: -w /etc/shadow -p wa -k shadow_access
```


## Post-Hardening Verification Checklist

```bash
# Telemetry disabled
ps aux | grep -E 'ubuntu-report|popularity-contest|abrt'
# Should return nothing

# Firewall enabled
sudo ufw status  # Ubuntu: Status: active
sudo firewall-cmd --list-all  # Fedora: shows drop zone

# Encryption enabled
sudo lsblk | grep crypt  # Shows encrypted volumes

# AppArmor/SELinux enabled
sudo aa-status | head -5  # Ubuntu: shows enforced profiles
getenforce  # Fedora: should return "Enforcing"

# Firefox hardened
# Visit about:config, verify privacy settings are set

# Unnecessary services disabled
systemctl list-units --state=running | wc -l  # Should be <20

# No world-readable sensitive files
find ~ -type f -perm /077 | wc -l  # Should return 0

# SSH hardened (if enabled)
grep "PasswordAuthentication no" /etc/ssh/sshd_config
```


## Security Hardening Maintenance

Monthly:
- Run security updates (automatic if enabled)
- Check firewall logs: `sudo journalctl -u ufw` or `sudo firewall-cmd --list-rich-rules`
- Review auth logs: `sudo journalctl -u auth`

Quarterly:
- Verify no new telemetry packages installed
- Check Firefox privacy settings haven't been reset
- Review AppArmor/SELinux denials: `sudo aa-logprof`

This hardened Linux desktop now refuses telemetry, limits network exposure, encrypts sensitive data, and sandboxes applications. The trade-off: slightly more complex to manage, but vastly more privacy-respecting.


## Related Articles

- [Pop Os Vs Fedora Vs Debian For Privacy Focused Linux Desktop](/privacy-tools-guide/pop-os-vs-fedora-vs-debian-for-privacy-focused-linux-desktop/)
- [Linux Kernel Hardening with sysctl](/privacy-tools-guide/linux-kernel-hardening-sysctl-guide)
- [Best VPN for Linux Desktop: A Developer Guide](/privacy-tools-guide/best-vpn-for-linux-desktop/)
- [Protonmail Bridge Setup For Desktop Email Clients Privacy Co](/privacy-tools-guide/protonmail-bridge-setup-for-desktop-email-clients-privacy-co/)
- [Linux Mint Privacy Setup Guide for Beginners](/privacy-tools-guide/linux-mint-privacy-setup-guide-beginners/)

{% endraw %}

