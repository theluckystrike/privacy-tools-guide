---
layout: default
title: "Encrypt Your Entire Digital Life: A Checklist"
description: "Complete guide to encrypting all data at rest and in transit. Covers device encryption, email, messaging, storage, passwords, and dead man's switches"
date: 2026-03-20
last_modified_at: 2026-03-20
author: theluckystrike
permalink: /encrypt-entire-digital-life--checklist/
categories: [guides, security]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Encryption is not paranoia—it's basic digital hygiene. Yet most people encrypt nothing. Your emails are readable by your provider. Your photos live unencrypted in cloud storage. Your passwords are managed by browsers. This guide walks through encrypting everything: devices, email, messages, files, passwords, and backups. It's neither impossible nor expensive. With 2 hours of setup time and free tools, you can achieve military-grade encryption across your digital life.

## The Encryption Pyramid

```
                    ▲ Most Important (highest impact)
                   ╱ ╲
                  ╱   ╲  Backups & Recovery
                 ╱─────╲
                ╱       ╲
               ╱         ╲ Cloud Storage & Syncing
              ╱───────────╲
             ╱             ╱ Email & Messaging
            ╱─────────────╱
           ╱             ╱ Passwords & Secrets
          ╱─────────────╱
         ╱             ╱ Device Encryption
        ╱─────────────╱
       ╱   Foundation  ╱ ▼ Least Important (quickest wins)
      ╱_______________╱
```

Start with the foundation and work upward. Device encryption gives you 80% security with 5 minutes of setup.

## Phase 1: Device Encryption (Foundation)

Device encryption ensures that if your laptop or phone is stolen, the thief sees gibberish, not your data.

### macOS: FileVault

**Status check:**
```
System Settings → Privacy & Security → FileVault
Click "Turn On FileVault"
```

**What happens:**
- Your entire drive gets encrypted with a strong key
- You'll be prompted for password at boot
- Background encryption happens automatically
- Takes 1-6 hours depending on drive size

**Backup your recovery key:**
```
FileVault shows: "Your recovery key is: XXXX-XXXX-XXXX-..."
Copy this to a safe place (Bitwarden, password manager, or paper)
Without this key, you cannot access your drive if you forget your password
```

**Performance impact:** Negligible (<1% CPU, no perceivable slowdown)

**Cost:** Free (included with macOS)

### Windows 11: BitLocker

**Enable BitLocker:**
```
Control Panel → System & Security → BitLocker Drive Encryption →
Turn On BitLocker
```

**What happens:**
- Entire C: drive encrypted
- Password required at boot (or Windows PIN)
- Microsoft stores recovery key in OneDrive (optional, configure carefully)

**Recovery key:**
```
BitLocker shows: "Your recovery key is XXXX-..."
Save to USB drive or offline storage
Do NOT rely on OneDrive storage unless you fully trust Microsoft
```

**Important:** Windows Home Edition doesn't have BitLocker. Use third-party:
- VeraCrypt (free, open-source)
- Axcrypt (free)

**Cost:** Free (Windows Pro+)

### Linux: LUKS Full-Disk Encryption

**Status check (if already encrypted during install):**
```bash
# Check if root partition is encrypted
sudo dmsetup status | grep crypt

# Output: crypt0: 0 XXXX crypt ...
# If output appears, you're encrypted ✓
```

**Enable if not encrypted (requires fresh install):**
```
During Ubuntu/Fedora install:
Check: "Encrypt the new installation for security"
Set strong passphrase (25+ characters)
```

**Cost:** Free (standard on most distributions)

### iOS: Automatic

iPhones are encrypted by default (since iOS 8). No action needed. Verify:
```
Settings → Face ID & Passcode → Enable "Face ID" or "Passcode"
(All data encrypted with key derived from your passcode)
```

### Android: Automatic

Most Android phones encrypt by default (since Android 5.0). Verify:
```
Settings → Security → Encryption
Status should show: "Phone encrypted"
```

**Force encrypt (if not auto-enabled):**
```
Settings → Security → Device Admin
Enable encryption (may take 1-4 hours)
```

## Phase 2: Passwords & Secrets Management

Unencrypted passwords are your biggest vulnerability. A hacked website reveals your password everywhere you use it.

### Use a Password Manager

**Recommended: Bitwarden (Open-Source, Free)**

```
Installation:
1. Download from bitwarden.com
2. Sign up with strong master password (30+ chars, random)
3. Save recovery code in secure location
4. Never, ever forget your master password
   (You cannot recover encrypted vault if you do)
```

**Or: 1Password (Commercial, $4.99/month)**

```
1Password advantages:
- Emergency access (trusted contact can access if you lose password)
- More polished UX
- Family plan ($7.99/month for 5 people)
```

**Setup process (Bitwarden):**

```
1. Master password: Generate 30+ character password
   Example: Tr0ub4dor&3*DFghjvC#JKLqwerty!999zx
   Store in offline location (paper notebook)

2. Import existing passwords:
   Settings → Tools → Import
   Select browser/Excel
   Upload file (Bitwarden decrypts at rest, deletes source)

3. Delete browser-saved passwords:
   Chrome: Settings → Autofill → Passwords → Delete all
   Safari: Passwords → Delete
   Firefox: About:preferences → Privacy → Saved passwords → Clear

4. Update critical passwords:
   Email, banking, cloud storage (these should be UNIQUE and STRONG)
```

**Cost:**
- Bitwarden: $10/year (optional paid cloud, free local sync)
- 1Password: $59.99/year
- LastPass: ~$36/year (but had 2022 breach, not recommended)

### Store Secrets Offline

Passwords for critical accounts (master password, encryption keys) should never be in any digital form:

```
Create a "Secrets Recovery Kit":
1. Purchase two fireproof safe deposit boxes (or home safe)
2. Create printout with:
   - Master password (Bitwarden/1Password)
   - Encryption keys (if using VeraCrypt, etc.)
   - Recovery codes (2FA backup codes)
   - Emergency contact instructions

3. Store in fireproof safe or deposit box
4. Give copy to trusted family member (separate location)
5. Update annually or after any password change
```

## Phase 3: Email Encryption (PGP/GPG)

Email is sent in plaintext across the internet. Anyone on the network path can read it. PGP encryption solves this.

### Generate PGP Keys

**Installation (macOS):**
```bash
brew install gnupg
brew install pinentry-mac
```

**Generate key pair:**
```bash
gpg --full-generate-key
```

**Prompted for:**
```
Key type: RSA and RSA (default)
Key size: 4096
Validity: 0 (never expires) or 1y (1 year, renew annually)
Real name: Your Name
Email: your.email@domain.com
Passphrase: Use strong password (different from master password)
```

**Export your public key (share with others):**
```bash
gpg --armor --export your.email@domain.com > my-public-key.asc
# Share this file publicly; others use it to encrypt emails to you
```

**Keep your private key safe:**
```bash
gpg --armor --export-secret-keys your.email@domain.com > my-private-key.asc
# Store this file in encrypted storage, never share
# Better: Don't export at all, keep it in GPG keyring
```

### Use PGP with Email Clients

**Thunderbird + Enigmail (Most User-Friendly):**

```
1. Download Thunderbird + Enigmail extension
2. Import your GPG key (already generated above)
3. Open email → Click "OpenPGP" → "Encrypt & Sign"
4. Sender's public key auto-fetches from keyserver
5. Email sent encrypted
```

**Manual (if not using integrated tools):**

```
1. Copy message in plaintext
2. Terminal: gpg --encrypt --armor --recipient email@example.com
3. Paste message
4. Press Ctrl+D twice
5. Encrypted output appears
6. Paste into email
```

### Key Verification (Critical!)

Without key verification, someone can intercept your public key and replace it with their own. Verify in person or via call:

```
When receiving someone's public key:
1. Get key fingerprint (16-character code)
   fingerprint = gpg --fingerprint email@example.com
   Example: E4AC B69F 5FAA F01C

2. Call them or meet in person
   "My key fingerprint is E4AC B69F 5FAA F01C"

3. They verify: "Yes, mine shows the same"

4. Mark as trusted
   gpg --edit-key email@example.com
   > trust
   > 4 (I trust fully)
   > quit
```

**Cost:** Free (GPG is open-source)

## Phase 4: Messaging Encryption

Messaging apps send billions of messages unencrypted daily. End-to-end encryption prevents your provider from reading your messages.

### Signal (Best for Encrypted Messaging)

```
Download Signal (iOS, Android, Desktop)
Sign up with phone number
All messages encrypted end-to-end by default
No account, no cloud backup (you can't access old messages after reinstall)
```

**Group encryption:**
```
Create group → Add members
Everyone can only read messages from members in the conversation
Server cannot read group content
```

**Cost:** Free (Open-source, funded by Signal Foundation)

### Element (Best for Self-Hosted)

```
Download Element (iOS, Android, Desktop)
Sign up on matrix.org or self-hosted server
All messages encrypted end-to-end
Can access message history across devices (unlike Signal)
```

**Configuration:**
```
1. Create account on matrix.org (or your own server)
2. All DMs encrypted to recipient's public key
3. Group chats encrypted for all members
4. Device verification (to prevent man-in-the-middle)
```

**Cost:** Free for matrix.org, or self-hosted

### WhatsApp (Good Enough)

```
Downloads WhatsApp (99% of people have it)
Encryption enabled by default since 2016
Better than nothing, but:
- Meta (Facebook) owns it, can see metadata
- Phone number is username (privacy leaky)
- Not open-source
```

**Use if:** Everyone you communicate with uses it, tradeoff is acceptable

### NOT Recommended

```
Do NOT use for sensitive communication:
- Email without PGP
- iMessage (encrypted, but Apple has master keys)
- Telegram (not encrypted by default, opt-in only)
- Facebook Messenger (unencrypted)
```

## Phase 5: Cloud Storage Encryption

Cloud storage (Dropbox, Google Drive, OneDrive) stores files unencrypted server-side. Use encrypted alternatives.

### Proton Drive (Best Balance)

```
Install Proton Drive
Sign up with Proton email or standalone account
Upload files → Encrypted server-side with your key
Download on any device → Auto-decrypted
```

**Architecture:**
```
Your device: Plain file
├─ Encrypt with your key
└─ Upload to Proton server
   (File stored encrypted)
```

**Cost:**
- Free: 5 GB
- Proton Plus: $11.99/month (500 GB Proton Drive + Mail + VPN)

**Sync setup:**
```
1. Download Proton Drive desktop app
2. Select local folder to sync
3. Files auto-encrypt on upload
4. Other devices auto-decrypt on download
```

### Sync.com (Alternative)

```
Install Sync.com
Sign up with email
Upload files → Encrypted end-to-end
Share links with password protection
```

**Cost:**
- Free: 5 GB
- Premium: $5.99/month (2 TB)

### Tresorit (If You Need European Privacy)

```
Based in Switzerland (strong privacy laws)
End-to-end encryption of all files
GDPR compliant
```

**Cost:** $11.50/month

### NOT Recommended

```
Unencrypted cloud storage:
- Dropbox (uses server-side encryption, but Dropbox has key access)
- Google Drive (encrypted in transit, plain on servers)
- OneDrive (encrypted in transit, plain on servers)
- iCloud (encrypted, but Apple has master key)
```

**If you must use these:** Add application-level encryption:
```
1. Encrypt files with VeraCrypt before uploading
2. Upload encrypted container to Google Drive
3. Download container, decrypt locally
```

## Phase 6: Email Service Replacement

Free email providers (Gmail, Yahoo) read your email to build ad profiles. Switch to private email:

### Proton Mail (Best Overall)

```
Sign up at protonmail.com
Get secure email address (username@proton.me)
All emails encrypted end-to-end (with other Proton users)
```

**Setup:**
```
1. Create Proton Mail account
2. Add to Outlook/Thunderbird for desktop access
3. Generate PGP key (Settings → Keys)
4. Share public key with contacts
5. Start receiving encrypted emails
```

**Cost:**
- Free: 500 MB
- Proton Plus: $11.99/month (500 GB)

### Tutanota (Alternative)

```
Open-source email encryption
End-to-end encrypted for all users
Proton Mail and Tutanota users can exchange encrypted emails
```

**Cost:**
- Free: 1 GB
- Premium: $2.99/month

## Phase 7: File Encryption for Backups

Backups are vulnerable (external drives, cloud storage). Encrypt them:

### VeraCrypt (Full Disk or Volumes)

**Create encrypted volume:**

```bash
# Create 5 GB encrypted container
veracrypt --text --create encrypted-backup.img

# Prompted:
# Volume size: 5 GB
# Password: 30+ character password
# Encryption: AES (default)
# Hash algorithm: SHA-512
# Filesystem: NTFS

# Mount volume:
veracrypt --text encrypted-backup.img /mnt/backup

# Use like normal folder, files auto-encrypt on write
cp -r ~/sensitive-files /mnt/backup/

# Unmount (files now encrypted):
veracrypt --text --dismount /mnt/backup
```

**Cost:** Free, open-source

### 7-Zip (File-by-File Encryption)

**Encrypt files:**
```bash
# Compress and encrypt
7z a -tzip -p mypassword archive.zip ~/documents/

# Creates archive.zip with AES-256 encryption
# Password required to extract
```

**Cost:** Free, open-source

## Phase 8: Backups & Recovery (Redundancy)

Encrypted backups are useless if you can't decrypt them. Build recovery:

### The 3-2-1 Backup Rule

```
3: Keep 3 copies of important data
2: On 2 different media types (internal drive + external + cloud)
1: Store 1 copy offsite (different physical location)

Example:
Copy 1: Internal SSD (encrypted)
Copy 2: External drive at home (encrypted)
Copy 3: Cloud storage overseas (encrypted)
```

### Create Recovery Kits

```
For each encrypted system, document:
1. Master password
2. Encryption key / passphrase
3. Software versions used
4. Instructions for recovery

Example:

RECOVERY KIT - Main Laptop (Created: 2026-03-20)
────────────────────────────────────
Device: MacBook Pro M3
OS: macOS 14.6
Encryption: FileVault

FileVault Recovery Key: XXXX-XXXX-XXXX-XXXX-XXXX
Master Password: [30-character password written on paper]

Backup location: External drive (2TB WD My Passport)
Backup encryption: VeraCrypt
Backup password: [30-character password written on paper]

Cloud backup: Proton Drive
Proton account: email@proton.me
Proton password: [in Bitwarden, recovery key stored separately]

Recovery instructions:
1. Boot from macOS recovery (Cmd+R)
2. Use FileVault Recovery Key from this document
3. Once logged in, mount encrypted external backup
4. Use VeraCrypt password from this document
5. Copy files to new drive

Created: 2026-03-20
Review date: 2026-09-20 (6 months)
```

## Phase 9: Two-Factor Authentication (2FA)

Encryption is useless if someone steals your password. 2FA prevents this:

### Hardware Security Keys (Strongest)

```
Purchase: YubiKey 5 ($45-60)
- Supports FIDO2, U2F, TOTP
- Works with most major services
- Requires physical key + biometric

Setup:
1. Download YubiKey Manager
2. Configure one key as primary, one as backup
3. Register key with each service:
   Email, password manager, cloud storage
4. Store backup key in safe deposit box
```

**Cost:** $45-60 per key

### Authenticator Apps (Good Alternative)

```
Download: Microsoft Authenticator or Google Authenticator
1. Scan QR code when enabling 2FA on service
2. Phone displays 6-digit code, refreshes every 30 seconds
3. Code required alongside password to log in

Important: Enable backup codes (printed, stored offline)
```

**Cost:** Free

### SMS 2FA (Weakest)

```
Service sends code via SMS text message
Vulnerable to SIM swap attacks (attacker convinces carrier to switch your number)
Use only as last resort if 2FA app not available
```

## Complete Encryption Checklist

```
PHASE 1: Device Encryption (1 hour)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
☐ macOS: Enable FileVault
  └─ Backup recovery key to password manager
☐ Windows: Enable BitLocker
  └─ Store recovery key on USB
☐ Linux: Full disk encrypted at install
☐ iOS: Enable Face ID + Passcode
☐ Android: Verify encryption enabled

PHASE 2: Passwords (30 minutes)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
☐ Download password manager (Bitwarden/1Password)
☐ Generate 30+ character master password
☐ Store master password offline (safe deposit box)
☐ Import all existing passwords
☐ Delete browser-saved passwords
☐ Update critical passwords (unique & strong)
☐ Enable 2FA on password manager

PHASE 3: Email (45 minutes)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
☐ Switch to Proton Mail (or Tutanota)
☐ Generate PGP key pair
☐ Export and share public key
☐ Set up email client (Thunderbird + Enigmail)
☐ Educate contacts about encrypted email
☐ Enable 2FA on email account

PHASE 4: Messaging (20 minutes)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
☐ Download Signal or Element
☐ Create account
☐ Add trusted contacts
☐ Verify security codes with frequent contacts
☐ Delete old messaging apps (or reduce use)

PHASE 5: Cloud Storage (30 minutes)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
☐ Switch to Proton Drive (or Sync.com)
☐ Set up desktop sync
☐ Move important files to encrypted cloud
☐ Delete old cloud accounts (or cease using)
☐ Enable 2FA on cloud account

PHASE 6: Backups (1 hour)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
☐ Create VeraCrypt encrypted external drive backup
☐ Schedule automatic backups (weekly)
☐ Document backup password (offline)
☐ Create offsite copy (cloud or secondary location)
☐ Create recovery kit document
☐ Store recovery kit in safe deposit box

PHASE 7: Two-Factor Authentication (30 minutes)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
☐ Purchase hardware security keys (2x YubiKey)
☐ Register key with password manager
☐ Register key with email account
☐ Register key with cloud storage
☐ Download and print backup codes
☐ Store backup codes offline

TOTAL TIME: ~3.5 hours
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Annual Maintenance

Set calendar reminders:

```
Quarterly:
- Test backup restoration (annually at least)
- Review password manager for weak passwords
- Check for any new 2FA methods offered by services

Semi-Annual:
- Rotate recovery key / backup passwords
- Update encryption software (VeraCrypt, etc.)
- Review cloud storage for unencrypted files

Annual:
- Full recovery kit document review and update
- Renewal of SSL certificates (if self-hosting email)
- Audit all third-party access (app passwords, API keys)
```


## Related Reading

- [Privacy Audit Checklist for SaaS Companies](/privacy-tools-guide/privacy-audit-checklist-for-saas-companies--gui/)
- [Privacy Setup For Stalking Victim Digital Prot](/privacy-tools-guide/privacy-setup-for-stalking-victim--digital-prot/)
- [Insurance Company Data Collection Privacy What Health.](/privacy-tools-guide/insurance-company-data-collection-privacy-what-health-life-auto/)
- [How To Test Vpn For Webrtc Leaks Testing Guide](/privacy-tools-guide/how-to-test-vpn-for-webrtc-leaks--testing-guide/)
- [Best Way to Encrypt Google Drive Files: A Developer Guide](/privacy-tools-guide/best-way-to-encrypt-google-drive-files/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
