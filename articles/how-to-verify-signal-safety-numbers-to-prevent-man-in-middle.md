---
---
layout: default
title: "How To Verify Signal Safety Numbers To Prevent Man"
description: "A technical guide for developers and power users on verifying Signal safety numbers. Learn to protect your encrypted communications from MITM attacks"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-verify-signal-safety-numbers-to-prevent-man-in-middle/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

In Signal, open a conversation with your contact, tap their name, and select "View Safety Number" to see a 60-digit code or QR code. Verify this in person or through a trusted video call against their device's displayed safety number—they see the same code. If the numbers match, your encryption keys are authentic and no MITM attack is occurring. You can scan each other's QR codes in the app to mark verification as complete, which sends a notification showing others in the chat that you've verified each other's identities.

## Table of Contents

- [Understanding Safety Numbers](#understanding-safety-numbers)
- [Accessing Safety Numbers](#accessing-safety-numbers)
- [Verifying Safety Numbers: The Process](#verifying-safety-numbers-the-process)
- [Automating Verification for Power Users](#automating-verification-for-power-users)
- [Understanding Attack Scenarios](#understanding-attack-scenarios)
- [When to Verify](#when-to-verify)
- [Common Pitfalls](#common-pitfalls)
- [Troubleshooting Mismatches](#troubleshooting-mismatches)
- [Advanced: Fingerprint Comparison](#advanced-fingerprint-comparison)

## Understanding Safety Numbers

Every Signal conversation generates a unique safety number based on the X3DH key agreement protocol and Double Ratchet algorithm. This number combines:

- Your device identity key
- Your contact's device identity key
- Session-specific ephemeral keys

The safety number changes when either party reinstalls Signal, adds a new device, or in some cases when keys are regenerated. This is a security feature—not a bug.

Safety numbers appear as 60-digit numbers displayed in groups:

```
12345 67890 12345 67890 12345 67890 12345 67890 12345 67890
```

Signal also generates a QR code for easier verification.

## Accessing Safety Numbers

### On Mobile

1. Open the conversation with your contact
2. Tap the conversation header (name or number)
3. Scroll down and tap "View safety number"
4. You'll see the 60-digit number and a QR code

### On Desktop (signal-cli or Desktop App)

If you're using the Signal Desktop client or signal-cli:

```bash
# List conversations with signal-cli
signal-cli listConversations

# Get detailed conversation info including safety numbers
signal-cli -o json receive | jq '.[] | select(.type == "typing")'
```

The Desktop client displays safety numbers in the same location as mobile: conversation header → "View safety number."

## Verifying Safety Numbers: The Process

Verification involves comparing the safety number through an independent channel—one that can't be intercepted by the same attacker targeting your Signal traffic.

### Step 1: Obtain the Safety Number

Get the safety number from your device as shown above. Both parties need to do this.

### Step 2: Choose Your Verification Channel

Select a channel that provides authentication independent of Signal:

**In-person verification**: Meet physically and compare numbers on each other's devices. This provides the highest security because the attacker would need to be physically present.

**Voice call (non-Signal)**: Call your contact on a regular phone or a different encrypted platform. Voice verification has limitations—call interception is possible but requires different capabilities than Signal traffic analysis.

**Another encrypted messenger**: Use a different end-to-end encrypted platform to exchange safety numbers. This assumes the alternative platform isn't also compromised.

**SMS with PGP**: For the paranoid, exchange safety numbers signed with PGP. This creates a verifiable paper trail.

### Step 3: Compare and Confirm

Both parties read their safety numbers aloud (in-person) or type them out (text-based channels). If the numbers match, your connection is secure—no MITM attacker has compromised the key exchange.

If numbers don't match, do not communicate sensitive information. The mismatch indicates either:
- A legitimate key change (new device, reinstall)
- An active attack

Contact your contact through a different channel to verify.

## Automating Verification for Power Users

For developers who want programmatic safety number verification, several approaches exist.

### Using signal-cli

Extract safety numbers via the API:

```bash
# Get account details including linked devices
signal-cli account

# Examine conversation details
signal-cli -o json send -m "test" +1234567890 | jq
```

Note: signal-cli doesn't directly expose safety numbers through CLI flags. You may need to inspect the local database.

### Parsing Signal's Local Database

Signal stores data locally in SQLCipher-encrypted databases. On Android, the path is:

```
/data/data/org.thoughtcrime.securesms/databases/signal.db
```

On Desktop (Linux):

```
~/.config/Signal/sql/db.sqlite
```

Query the database directly (requires key extraction):

```python
import sqlite3

# Signal stores safety numbers in the recipient table
conn = sqlite3.connect('/path/to/signal.db')
cursor = conn.cursor()

# Get recipient identity info
cursor.execute('''
    SELECT recipient_id, identity_key, identity_key_verified
    FROM recipient
    WHERE recipient_id = ?
''', (contact_id,))

for row in cursor.fetchall():
    print(f"Recipient ID: {row[0]}")
    print(f"Identity Key (hex): {row[1].hex()}")
    print(f"Verified: {row[2]}")
```

The actual safety number calculation involves the X3DH result and Double Ratchet state. Replicating this precisely requires studying Signal's open-source implementation.

### Building a Verification Script

Create a script that monitors for safety number changes:

```bash
#!/bin/bash
# monitor-safety.sh
# Usage: ./monitor-safety.sh <phone-number>

CONTACT="$1"
DB_PATH="$HOME/.config/Signal/sql/db.sqlite"

if [ -z "$CONTACT" ]; then
    echo "Usage: $0 <phone-number>"
    exit 1
fi

# Query current identity state
sqlite3 "$DB_PATH" "
SELECT
    r.phone_number,
    hex(i.identity_key) as identity_key,
    i.verified_time
FROM recipient r
JOIN identity_key i ON r.id = i.recipient_id
WHERE r.phone_number = '$CONTACT'
;" 2>/dev/null

echo "If the identity key changes unexpectedly, investigate immediately."
echo "Safe keys should only change after device changes or reinstalls."
```

## Understanding Attack Scenarios

Knowing why safety number verification matters requires understanding the attack vectors.

### The Man-in-the-Middle Attack

In a MITM attack on Signal, the attacker intercepts the key exchange between you and your contact. They establish two separate encrypted sessions—one with you, one with your contact—decrypting, reading, and re-encrypting all messages.

Without safety number verification:
- Signal shows "end-to-end encrypted" for both sessions
- Messages flow normally
- You have no indication of compromise

With safety number verification:
- Your safety numbers won't match your contact's
- The mismatch alerts you to the attack
- You can confirm via independent channel

### The Threat Model

Safety number verification protects against:
- State-level adversaries who can intercept network traffic
- Compromised telecommunications providers
- Sophisticated hackers with network positioning

It doesn't protect against:
- Compromised devices (malware on your phone)
- Social engineering (attacker impersonating your contact)
- Signal server compromise (they can't read messages, but could deny service)

## When to Verify

Verify safety numbers in these scenarios:

1. **Initial contact**: When starting a sensitive conversation for the first time
2. **After device changes**: When you or your contact gets a new phone
3. **After reinstalls**: When Signal is reinstalled
4. **Periodically**: For high-security relationships, verify quarterly
5. **Before sensitive exchanges**: Before sharing credentials, financial info, or classified data

## Common Pitfalls

**Relying only on "verified" badges**: Signal marks contacts as verified if you've previously confirmed safety numbers. If those numbers change, Signal alerts you—but only if you notice.

**Ignoring warnings**: Signal shows prominent warnings when safety numbers change. Don't dismiss these.

**Verifying through Signal**: Don't verify safety numbers through Signal itself. Use an independent channel.

**Weak verification channels**: Avoid verifying through email or unencrypted SMS—these can be intercepted.

## Troubleshooting Mismatches

When safety numbers don't match:

1. **Confirm separately**: Contact your contact through a known-good channel (phone call, in-person)
2. **Check for legitimate causes**: Did they get a new phone? Reinstall Signal?
3. **Re-establish verification**: After confirming legitimacy, verify the new safety numbers
4. **Document the change**: In high-security contexts, log when and why keys changed

## Advanced: Fingerprint Comparison

For maximum security, compare identity key fingerprints rather than just safety numbers. This provides cryptographic certainty:

```bash
# Extract identity keys from Signal database
# Then compare using SHA-256

echo "Your identity key fingerprint:"
echo -n "$YOUR_KEY" | sha256sum | cut -d' ' -f1

echo "Contact's identity key fingerprint:"
echo -n "$THEIR_KEY" | sha256sum | cut -d' ' -f1
```

Matching fingerprints prove you're using the same identity keys that generated the safety number.

## Frequently Asked Questions

**How long does it take to verify signal safety numbers to prevent man?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Can I adapt this for a different tech stack?**

Yes, the underlying concepts transfer to other stacks, though the specific implementation details will differ. Look for equivalent libraries and patterns in your target stack. The architecture and workflow design remain similar even when the syntax changes.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [How To Use Abine Blur For Masked Emails Phone Numbers And Cr](/privacy-tools-guide/how-to-use-abine-blur-for-masked-emails-phone-numbers-and-cr/)
- [How To Use Virtual Credit Card Numbers From Privacy Com For](/privacy-tools-guide/how-to-use-virtual-credit-card-numbers-from-privacy-com-for-/)
- [Crypto Dead Man Switch Services That Transfer Wallet Access](/privacy-tools-guide/crypto-dead-man-switch-services-that-transfer-wallet-access-/)
- [Set Up a Dead Man's Switch Email That Sends Credentials If](/privacy-tools-guide/how-to-set-up-dead-mans-switch-email-that-sends-credentials-/)
- [Set Up Dead Man's Switch Using Cron Job to Release Encrypted](/privacy-tools-guide/how-to-set-up-dead-mans-switch-using-cron-job-to-release-enc/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
