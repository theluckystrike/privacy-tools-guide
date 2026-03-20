---

layout: default
title: "How to Prepare VPN and Security Tool Credentials for Family Member Taking Over Accounts"
description: "A technical guide for developers and power users on securely transferring VPN, password manager, and security tool access to family members. Includes code examples and best practices."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-prepare-vpn-and-security-tool-credentials-for-family-/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
---

{% raw %}

If you manage your own VPN infrastructure, self-hosted security tools, or encrypted services, you probably haven't thought about what happens when someone else needs to take over. Unlike logging into a Netflix account, transferring access to VPNs, password managers, and security tools requires careful planning. The goal is to ensure continuity of access while maintaining security—not just for you, but for anyone who needs to manage these systems after you're gone or unable to do so.

This guide covers the technical steps for preparing VPN credentials, password manager access, 2FA backups, and self-hosted security tool administration for family members who may need to take over.

## Documenting VPN Access

When preparing VPN credentials for transfer, you need to consider whether you're using a commercial VPN service or a self-hosted solution like WireGuard, OpenVPN, or Outline.

### Commercial VPN Services

For commercial VPNs like Mullvad, Proton VPN, or IVPN, the account credentials are typically tied to an email address and payment method. Document the following:

- Email address used for the VPN account
- Account ID or username
- Payment method on file (for renewal purposes)
- Subscription status and renewal date

Most modern privacy-focused VPNs support account recovery through email. Make sure your family member knows which email address receives the VPN invoices and account communications.

### Self-Hosted WireGuard Configurations

If you run your own WireGuard VPN on a VPS, the configuration is more complex. Each WireGuard client needs a private key and corresponding public key registered on the server. Here's how to export and document a client configuration:

```bash
# On the WireGuard server, list active peers
sudo wg show wg0 peers

# Export a specific client's configuration
sudo wg show wg0 peer <public-key>
```

Create a comprehensive configuration file for your family member:

```ini
# family-member-vpn.conf
[Interface]
PrivateKey = <your-family-members-private-key>
Address = 10.0.0.5/24
DNS = 1.1.1.1

[Peer]
PublicKey = <servers-public-key>
Endpoint = your-vpn-server.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Store this configuration securely—preferably in a password manager or encrypted storage that your family member can access.

## Password Manager Emergency Access

Most password managers have built-in emergency access or vault inheritance features. This is the single most important security tool to prepare for transfer.

### Bitwarden Emergency Access

Bitwarden offers a native emergency access feature. Your designated family member can request access to your vault, and you'll have a waiting period (you configure 1-7 days) to deny the request before automatic release:

```bash
# Bitwarden CLI command to set up emergency access (requires web vault for initial setup)
# In the web vault:
# 1. Go to Settings > Emergency Access
# 2. Enter your family member's Bitwarden email
# 3. Set the waiting period (recommended: 3-7 days)
# 4. Confirm the invitation
```

The family member must also have a Bitwarden account and enable emergency access from their own account settings.

### KeePassXC Database Transfer

For KeePassXC users, the database file itself needs to be transferred along with the master password. Unlike cloud-based managers, KeePassXC offers no automatic inheritance mechanism. You'll need to:

1. Store the KeePassXC database file in a secure, accessible location (encrypted USB drive, safe deposit box, or encrypted cloud storage)
2. Document the master password separately—ideally in a sealed envelope or with an attorney
3. Consider splitting the password using Shamir's Secret Sharing if you want multiple family members to collaborate on access

### 1Password Families

1Password offers a built-in families plan where you can add family members directly. They inherit access to shared vaults automatically. If you're on an individual plan, convert to a families plan and add your family member as an adult member:

```
# 1Password Families structure:
- Family Organizer (you): Full access, can manage other members
- Adult Member (family member): Access to shared vaults
- Optional: Children can have restricted access
```

## Two-Factor Authentication Backup Transfer

2FA is critical for security, but it creates a single point of failure if not properly documented. When transferring 2FA access, you have several options depending on the authenticator app used.

### TOTP Seed Transfer

Time-based one-time passwords (TOTP) are generated from a shared secret seed. If your family member needs to receive codes from the same accounts, you must transfer the TOTP seeds, not just the screenshots of QR codes.

```bash
# Extract TOTP seeds from various authenticator apps

# For andotp (Android):
# Export encrypted backup, decrypt with your password
# Seeds are stored in JSON format

# For 2FAS (iOS/Android):
# Export to JSON, look for "secret" fields
```

Document each TOTP seed in a structured format:

```json
{
  "service": "GitHub",
  "account": "your-email@example.com",
  "totp_seed": "JBSWY3DPEHPK3PXP",
  "issuer": "GitHub",
  "algorithm": "SHA1",
  "digits": 6,
  "period": 30
}
```

Store this file encrypted, separate from where you store the decryption key.

### YubiKey Configuration

If you use hardware security keys like YubiKeys for 2FA, you need to register a second key with your accounts. Not all services support multiple security keys, but many do:

- **GitHub**: Settings > Security > Security keys > Add
- **Google**: Settings > Security > How you sign in > 2-Step Verification > Add security key
- **Password managers**: Most support multiple hardware tokens

Register a backup YubiKey to your important accounts and document its location separately from your primary key.

## Self-Hosted Security Tools

If you run security tools on your own infrastructure—Pi-hole for DNS filtering, a self-hosted VPN, or a firewall configuration—document the administrative access thoroughly.

### SSH Key Administration

For servers accessed via SSH, ensure your family member has their own SSH key provisioned:

```bash
# Generate a new SSH key for your family member
ssh-keygen -t ed25519 -C "family-member@email.com"

# Add their public key to the server
ssh-copy-id -i ~/.ssh/family_member_key.pub user@your-server.com

# On the server, restrict their key to specific commands if needed
# In ~/.ssh/authorized_keys:
command="/usr/bin/sudo /usr/bin/systemctl status vpn-service",no-agent-forwarding,no-port-forwarding ssh-ed25519 AAAA...
```

### Firewall and VPN Server Documentation

Create a runbook for your family member that includes:

```bash
#!/bin/bash
# family-vpn-management.sh
# Common commands for managing the family VPN

# Check VPN service status
sudo systemctl status wg-quick@wg0

# View active connections
sudo wg show

# Restart the VPN service
sudo systemctl restart wg-quick@wg0

# Check firewall rules
sudo iptables -L -n -v

# View recent connection logs
sudo journalctl -u wg-quick@wg0 -n 50 --no-pager
```

Store this runbook alongside your other documentation, and walk your family member through the basics before they need to use it.

## Structured Credential Handoff

Rather than scattering credentials across password managers, text files, and USB drives, create a structured handoff document. Use age-encryption or GPG to encrypt the document, then store it in a location your family member can access:

```bash
# Encrypt credentials using age (modern GPG alternative)
age -p -o credentials.age <<EOF
VPN Configuration:
- Server: vpn.yourdomain.com
- Username: admin
- WireGuard Config: See attached file

Password Manager:
- Service: Bitwarden
- Emergency Contact: family-member@email.com

2FA Backup:
- TOTP Seeds: See encrypted seeds.json
- Backup YubiKey: Safe deposit box, Box #3

Server Access:
- SSH Key Location: USB drive in filing cabinet
- Management Script: See family-vpn-management.sh
EOF
```

## Testing the Handoff

Before you need the transfer to work, test it. Have your family member attempt to:

1. Log into the password manager using emergency access
2. Connect to the VPN using the documented configuration
3. Access one server via SSH using their provisioned key
4. Generate a TOTP code from the transferred seed

Document any friction points and refine your documentation accordingly. The goal is a smooth transition when it becomes necessary.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
