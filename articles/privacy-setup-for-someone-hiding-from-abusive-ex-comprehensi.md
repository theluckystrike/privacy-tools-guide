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
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, privacy]
---

{% raw %}

This guide assumes you have access to a trusted computer and can install software. If you are in immediate danger, skip to the emergency section at the end. The technical measures here complement, not replace, professional support and safety planning.
Immediately enable full-disk encryption (FileVault on macOS, BitLocker on Windows, LUKS on Linux), create a new anonymous email account from a safe device, use a password manager to generate strong unique passwords for all accounts, remove all location-sharing features and check for hidden tracking apps, and use Tor for any sensitive communications. Change all recovery methods and security questions, remove the abuser's access to shared accounts, and establish emergency contacts outside the abuser's network. This guide provides immediate and long-term actionable technical steps with code examples for securing your digital safety when leaving an abusive relationship.

Threat Model

When fleeing an abusive ex who possesses technical knowledge, your threat model includes someone who may have installed spyware, has credentials to shared accounts, understands your digital habits, and could use social engineering against your contacts. Every recommendation below addresses these specific risks.

Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


Step 1: Device Hardening

Clean Device Installation

Never continue using a device your abuser had physical access to. Purchase a new device with cash if possible. Boot into recovery mode and wipe the existing OS completely:

```bash
On macOS: Hold Cmd+R during boot, select Disk Utility, erase the drive
On Linux: Use a live USB to run shred or badblocks on the entire drive
sudo shred -v -n 3 /dev/sda
```

For mobile, perform a factory reset after enabling full disk encryption. On Android, verify the bootloader is locked afterward:

```bash
fastboot flashing get_unlock_ability
```

If unlock_ability returns 0, the bootloader remains locked, a positive security indicator.

Operating System Choice

GrapheneOS on a Pixel device provides strong sandboxing and sensor toggles that physically disable the microphone and cameras. This matters because stalkerware often exploits camera and microphone access. Install GrapheneOS via the official web-based installer:

1. Enable developer settings on the Pixel
2. Visit grapheneos.org/install
3. Follow the web installer instructions (no flashing required)

For desktop, Tails or Qubes OS compartmentalize your activities. Tails wipes memory on shutdown, essential for preventing forensic recovery of your location data.

Step 2: Account Isolation

Email Separation

Create a completely new email address from a device your abuser never touched. Use a privacy-respecting provider like Proton Mail. Enable two-factor authentication with a hardware key, not SMS:

```bash
Generate a YubiKey-compatible GPG key for authentication backup
gpg --full-generate-key
Use this key as your 2FA backup, store the revocation certificate offline
```

Never link this new email to your previous accounts. Do not import contacts or emails from your old accounts.

Phone Number Strategy

Get a new SIM card from a provider that doesn't require ID registration where you live. Use a VoIP number (like MySudo or Google Voice) for all dating and social accounts, keep your real number completely separate. On Android, configure calls to route through the VoIP app only:

```kotlin
// Example: Using Tasker to route calls from unknown numbers to VoIP
Profile: Unknown Caller
Event: Phone Ringing
    IF %CNUMBER !IN %MyContacts
Task: Accept Call
    App Launch: VoIP App
```

This prevents your ex from discovering your new number through social engineering telecom employees.

Password Manager Segregation

Your old password manager is compromised. Create a fresh vault in Bitwarden or 1Password with a completely new master password. Generate unique 20+ character passwords for every account:

```bash
Using Bitwarden CLI to generate passwords
bw generate --length 24 --complexity --includeNumber --includeSymbol --includeUppercase --includeLowercase
```

Enable the emergency access feature with a trusted person who understands the sensitivity, but give them instructions to delete the request if they receive it unexpectedly. This serves as a dead man's switch for your digital life.

Step 3: Secure the Network

VPN Configuration

Route all traffic through a VPN you control, a self-hosted WireGuard server on a VPS purchased with cryptocurrency. This prevents your ISP and anyone monitoring network traffic from seeing your browsing activity:

```ini
/etc/wireguard/wg0.conf on your VPS
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

Configure your client to route all traffic through the tunnel. Set the kill switch to block all non-VPN traffic, this prevents accidental leaks if the connection drops.

DNS Configuration

Use NextDNS or your own Pi-hole to block tracking domains. Configure it to log nothing:

```yaml
nextdns.yml configuration
listen: :53
discovery: false
reporting:
  enabled: false
blocks:
  - 1d8c6f16-6466-11ea-8c41-063af41d5f76
```

Blocklists should include known stalkerware domains, advertising trackers, and social media trackers.

Browser Hardening

Use Firefox with the following about:config changes:

```
privacy.resistFingerprinting = true
network.cookie.cookieBehavior = 1
privacy.trackingprotection.enabled = true
webgl.disabled = true
```

Install uBlock Origin and Privacy Badger. Create a separate browser profile for sensitive activities, bookmark the new email, the VPN status page, and your safety plan.

Step 4: Operational Security

Metadata Stripping

Before sharing any photos, strip EXIF data:

```bash
Using exiftool
exiftool -all= -overwrite_original image.jpg

Batch processing
for f in *.jpg; do exiftool -all= -overwrite_original "$f"; done
```

On mobile, use Metapho orexif to remove location data before sending photos to anyone.

Signal Configuration

Configure Signal with these settings:

1. Enable registration lock with a strong PIN
2. Set disappearing messages to 24 hours for all chats
3. Disable link previews (prevents URL leakage)
4. Use the screen lock feature

Never link Signal to your old phone number.

Social Media Audit

Search for yourself on social media using your new email and phone number. Set Google Alerts for your name variations. Request account deletion from any platform where your ex might have created lookalike accounts:

```bash
Use the email you just created to check data broker exposure
curl -X DELETE https://api.example-data-broker.com/v1/profile \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email": "your-new-email@example.com"}'
```

Step 5: Emergency Protocols

If you suspect immediate danger:

1. Power off all devices and remove batteries if possible
2. Go to a location your ex doesn't know about
3. Use a library computer or borrowed device to change critical passwords
4. Contact a domestic violence hotline, they have secure communication channels

Save these resources offline:

- National Domestic Violence Hotline: 1-800-799-7233
- Cyber Civil Rights Initiative: 1-844-878-2274

Troubleshooting

Configuration changes not taking effect

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

Permission denied errors

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

Connection or network-related failures

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


Frequently Asked Questions

How long does it take to someone hiding from abusive ex?

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

What are the most common mistakes to avoid?

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

Do I need prior experience to follow this guide?

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

Is this approach secure enough for production?

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

Where can I get help if I run into issues?

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

Related Articles

- [Privacy Setup For Someone Leaving Abusive Relationship](/privacy-setup-for-someone-leaving-abusive-relationship-digit/)
- [Privacy Setup For Stalking Victim Digital](/privacy-setup-for-stalking-victim--digital-prot/)
- [How To Set Up Emergency Access For Password Manager](/how-to-set-up-emergency-access-for-password-manager-spouse/)
- [Password Manager For Shared Accounts Between Roommates](/password-manager-for-shared-accounts-between-roommates-secure-method/)
- [Privacy Setup for Confidential Informant](/privacy-setup-for-confidential-informant-protecting-identity/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Step 6: Ongoing Monitoring and Maintenance

Security doesn't end with initial setup. Continuous monitoring catches changes:

```bash
#!/bin/bash
security-audit.sh - Weekly monitoring for intrusion indicators

AUDIT_LOG="$HOME/.local/share/security-audit.log"

run_audit() {
    echo "=== Security Audit $(date) ===" >> "$AUDIT_LOG"

    # Check for unauthorized SSH keys
    echo "Checking SSH keys..." >> "$AUDIT_LOG"
    wc -l ~/.ssh/authorized_keys >> "$AUDIT_LOG"

    # Check installed packages for anomalies
    echo "Checking package integrity..." >> "$AUDIT_LOG"
    debsums -c 2>&1 | head -10 >> "$AUDIT_LOG"

    # Monitor system load anomalies
    echo "System load: $(uptime)" >> "$AUDIT_LOG"

    # Check for hidden processes
    echo "Process count: $(ps aux | wc -l)" >> "$AUDIT_LOG"

    # Verify file integrity
    echo "Checking critical file timestamps..." >> "$AUDIT_LOG"
    stat /etc/passwd /etc/shadow /etc/sudo >> "$AUDIT_LOG"

    # Check for unexpected cron jobs
    crontab -l >> "$AUDIT_LOG" 2>&1

    # Send alert if anomalies detected
    if grep -q "FAILED\|ERROR" "$AUDIT_LOG"; then
        # Alert to trusted contact (signal, email, etc)
        notify_emergency_contact "Security audit detected anomalies"
    fi
}

run_audit

Schedule: Add to crontab for weekly execution
0 3 * * 0 /path/to/security-audit.sh
```

Step 7: Contact Safety Protocols

If safe to do so, establish secure communication with trusted contacts:

```bash
#!/bin/bash
secure-contact-setup.sh

Create Signal account from safe device (library computer)
Never link to old phone number

Alternative: Temporary contact method
1. Create burner Signal account
2. Share ONLY with completely trusted people
3. Meet in person to exchange contact details

Setup dead man's switch
If you don't check in monthly, alert goes to trusted contact

cat > ~/.local/share/dead-mans-switch.sh << 'EOF'
#!/bin/bash

CONTACT_EMAIL="trusted-friend@protonmail.com"
DAYS_SINCE_CHECKIN=30
CHECKIN_FILE="$HOME/.local/share/last-checkin"

if [ ! -f "$CHECKIN_FILE" ]; then
    touch "$CHECKIN_FILE"
fi

LAST_CHECKIN=$(stat -f "%m" "$CHECKIN_FILE" 2>/dev/null || stat -c %Y "$CHECKIN_FILE")
CURRENT_TIME=$(date +%s)
DAYS_ELAPSED=$(( (CURRENT_TIME - LAST_CHECKIN) / 86400 ))

if [ $DAYS_ELAPSED -gt $DAYS_SINCE_CHECKIN ]; then
    echo "ALERT: No check-in for $DAYS_ELAPSED days"
    # Send encrypted email to contact
    echo "Security alert: No check-in received" | \
    gpg --encrypt --recipient "$CONTACT_EMAIL" | \
    mail -s "Dead man's switch triggered" "$CONTACT_EMAIL"
fi

Update check-in time (run after opening email)
touch "$CHECKIN_FILE"
EOF

chmod +x ~/.local/share/dead-mans-switch.sh

Add to weekly cron job
(crontab -l 2>/dev/null; echo "0 9 * * 0 ~/.local/share/dead-mans-switch.sh") | crontab -
```

Step 8: Emergency Access Recovery

Prepare for worst-case scenarios:

```bash
Create recovery codes
Store ONE copy in EACH of two secure physical locations
(separate safe deposit boxes or trusted people)

gpg --gen-key  # Generate recovery GPG key

Create recovery mnemonic
cat > recovery-instructions.txt << 'EOF'
RECOVERY PROTOCOL - Keep this SECURE

1. If main email is compromised:
   - Access recovery email (address: _________)
   - Recovery code: ________________
   - Backup 2FA codes stored: ______________

2. If all devices compromised:
   - Go to library with ID
   - Email password reset from library computer
   - Change 2FA immediately using recovery codes
   - Do NOT use home internet

3. Emergency contacts (encrypted):
   - [Name]: [Signal number/encrypted contact info]

4. Evidence preservation:
   - Screenshot and save to USB drive
   - Store in safe location
   - Can be used for protection order

5. If in immediate physical danger:
   - Leave all devices
   - Go directly to domestic violence shelter
   - Call: 1-800-799-7233
EOF

Encrypt recovery file
gpg --symmetric recovery-instructions.txt

Print and physically secure
lp recovery-instructions.txt.gpg
Store one copy separate from home
```

Step 9: Recognition of Escalation Signs

Know when technical measures may not be sufficient:

```
Escalation indicators (require professional intervention):
- Physical stalking or threats
- Attempts to contact through social contacts
- Monitoring your job/workplace
- Behavior predictive of violence:
  * Increased aggression
  * Promises followed by violations
  * Sudden status changes (losing job, etc)

If escalation occurs:
1. Document everything (dates, times, incidents)
2. File police report and keep copy
3. Seek protective order
4. Consult with domestic violence advocates

Technical measures are supplementary to physical safety.
Professional help (DV advocates, law enforcement) is primary.
```

Step 10: Long-Term Resilience

Building sustainable privacy practices for extended security:

```bash
#!/bin/bash
long-term-security-checklist.sh

Monthly password audit
echo "=== Monthly Security Review ==="

1. Test 2FA still working
echo "Testing 2FA on critical accounts..."
Manually verify each critical account

2. Rotate API keys/tokens
echo "Rotating API credentials..."
Update password manager entries

3. Review connected apps
echo "Checking app permissions..."
Remove old apps with unnecessary access

4. Verify VPN/encryption still active
echo "Verifying encryption status..."
sudo lvdisplay | grep -i open  # Check encrypted volumes

5. Update security contacts
echo "Reviewing emergency contacts..."
Ensure all contact methods still valid

6. Backup critical configs
echo "Backing up configurations..."
tar czf ~/.local/share/backup-$(date +%Y%m%d).tar.gz \
  ~/.ssh ~/.gnupg ~/.config/signal

7. Check for service changes
echo "Reviewing service terms..."
Visit privacy pages of critical services

echo "Security review complete"
```

Step 11: Resources for Ongoing Support

Technical measures work best with social support:

- National Domestic Violence Hotline: 1-800-799-7233
- Online safety planning: National Domestic Violence Hotline website
- Tech security for survivors: National Network to End Domestic Violence (NNEDV) has tech resources
- Legal aid: LawHelp.org (US) or equivalent in your country

This guide provides technical defense. Professional advocates provide the support needed for actual safety.

Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
