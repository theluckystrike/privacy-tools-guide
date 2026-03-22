---
layout: default
title: "How to Detect Phishing Emails with Headers"
description: "Read email headers to detect spoofed senders, verify SPF, DKIM, and DMARC results, and identify malicious routing in suspected phishing messages"
date: 2026-03-22
author: theluckystrike
permalink: /detect-phishing-emails-headers-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Detect Phishing Emails with Headers

Email headers contain a full audit trail — every server the message passed through, authentication results, and the actual sender identity. Most email clients hide them by default, but a 30-second header analysis can confirm whether a suspicious message is legitimate or a spoof.

## Table of Contents

- [How to Access Email Headers](#how-to-access-email-headers)
- [The Key Fields to Check](#the-key-fields-to-check)
- [Parsing Headers Programmatically](#parsing-headers-programmatically)
- [Quick Reference: Red Flags in Headers](#quick-reference-red-flags-in-headers)
- [Advanced Email Forensics](#advanced-email-forensics)
- [Integration with Email Clients](#integration-with-email-clients)
- [Industry-Specific Phishing Patterns](#industry-specific-phishing-patterns)
- [Related Reading](#related-reading)

## How to Access Email Headers

**Gmail:** Open the email → three dots menu → "Show original"

**Outlook (Web):** Three dots → View → View message source

**Apple Mail:** View → Message → All Headers (or Cmd+Shift+H)

**Thunderbird:** View → Message Source (Ctrl+U)

**Command line (with mbox or .eml file):**

```bash
# Display headers only
head -100 suspicious.eml

# Or parse with Python
python3 -c "
import email, sys
msg = email.message_from_file(open('suspicious.eml'))
for key in ['From', 'Reply-To', 'Return-Path', 'Received-SPF',
            'DKIM-Signature', 'Authentication-Results', 'X-Originating-IP']:
    if msg[key]:
        print(f'{key}: {msg[key][:200]}')
"
```

## The Key Fields to Check

### 1. From vs. Return-Path vs. Reply-To

```
From: PayPal Security <security@paypal.com>
Return-Path: <bounce@mail-provider-xyz.net>
Reply-To: response@totally-different-domain.com
```

- **From** — what you see in your client. Trivially spoofed.
- **Return-Path** — where bounces go. Usually the actual sending domain.
- **Reply-To** — if set and different from From, replies go here. Major red flag.

A legitimate PayPal email will have `Return-Path` ending in `@paypal.com`. If it ends in a random domain, the From address is spoofed.

### 2. Received Chain

Email headers stack from bottom to top — the first server that received the message is at the bottom.

```
Received: from mail.paypal.com (mail.paypal.com [173.0.84.2])
        by mx.google.com with ESMTPS id ...

Received: from smtp-internal.paypal.com (smtp-internal.paypal.com [10.0.0.15])
        by mail.paypal.com with ESMTP id ...
```

Read from the bottom up. The first hop should originate from the claimed domain. Check the IP addresses:

```bash
# Look up the originating IP
dig -x 185.234.219.45 +short
# random-vps-provider.net

# Check if it's on spam/malware blacklists
curl "https://api.abuseipdb.com/api/v2/check?ipAddress=185.234.219.45" \
  -H "Key: YOUR_ABUSEIPDB_API_KEY" \
  -H "Accept: application/json" | jq '.data.abuseConfidenceScore'
```

### 3. SPF (Sender Policy Framework)

SPF checks whether the sending server's IP is authorized by the domain's DNS records.

```
Received-SPF: pass (google.com: domain of security@paypal.com designates 173.0.84.2 as permitted sender)
```

Status meanings:
- **pass** — IP is authorized by the domain's SPF record (legitimate)
- **fail** — IP is explicitly not allowed (strong spam signal)
- **softfail** — IP is not listed but the domain uses `~all` (soft failure)
- **neutral** — domain doesn't express policy
- **none** — no SPF record exists

```bash
# Check a domain's SPF record
dig TXT paypal.com | grep spf
# "v=spf1 include:pp.spf.paypal.com include:3rdparty.paypal.com ~all"

# Check if a specific IP passes SPF
# Online tool: https://www.spf-record.com/spf-lookup
# or use pyspf
pip install pyspf
python3 -c "import spf; print(spf.check2(i='185.234.219.45', s='security@paypal.com', h='mail.attacker.com'))"
```

### 4. DKIM (DomainKeys Identified Mail)

DKIM adds a cryptographic signature to the message body and headers. Modification in transit breaks the signature.

```
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed; d=paypal.com; s=paypalmail;
    h=from:to:subject:date:message-id;
    bh=abc123...;
    b=MEQCIB...
```

Key fields:
- `d=` — the signing domain (must match or align with From domain)
- `s=` — the selector (used to look up the public key in DNS)
- `b=` — the signature

```
Authentication-Results: mx.google.com;
    dkim=pass header.i=@paypal.com header.s=paypalmail header.b=MEQCIB;
    spf=pass (google.com: ...) smtp.mailfrom=security@paypal.com;
    dmarc=pass (p=REJECT) header.from=paypal.com
```

A phishing email impersonating PayPal cannot produce a valid DKIM signature for `@paypal.com` because it does not have PayPal's private key.

```bash
# Look up DKIM public key manually
dig TXT paypalmail._domainkey.paypal.com
```

### 5. DMARC

DMARC ties SPF and DKIM together and tells receiving servers what to do with failures.

```
Authentication-Results: dmarc=fail (p=REJECT) header.from=paypal.com
```

DMARC results:
- **pass** — SPF or DKIM passed and the domain aligns with From
- **fail** — both SPF and DKIM failed alignment
- **bestguesspass** — DMARC record not found, best guess

When DMARC fails for a domain like `paypal.com` that publishes `p=REJECT`, the message should have been rejected by your mail server — if you received it, something is wrong with your mail setup or the email is from a lookalike domain.

```bash
# Check DMARC policy for a domain
dig TXT _dmarc.paypal.com
# "v=DMARC1; p=reject; rua=mailto:d@rua.agari.com; ..."
# p=reject means any failing message should be rejected outright
```

## Parsing Headers Programmatically

```python
#!/usr/bin/env python3
# analyze-email-headers.py

import email
import sys
import re

def analyze(filepath):
    with open(filepath, 'r', errors='replace') as f:
        msg = email.message_from_file(f)

    print("=== SENDER ANALYSIS ===")
    print(f"From:        {msg.get('From', 'NOT PRESENT')}")
    print(f"Reply-To:    {msg.get('Reply-To', 'not set')}")
    print(f"Return-Path: {msg.get('Return-Path', 'NOT PRESENT')}")

    print("\n=== AUTHENTICATION ===")
    auth = msg.get('Authentication-Results', '')
    for proto in ['spf', 'dkim', 'dmarc']:
        match = re.search(rf'{proto}=(\S+)', auth, re.IGNORECASE)
        result = match.group(1) if match else 'NOT FOUND'
        flag = "OK" if result == 'pass' else "FAIL"
        print(f"{proto.upper()}: {result} [{flag}]")

    print("\n=== RECEIVED CHAIN (bottom to top = first to last hop) ===")
    received = msg.get_all('Received', [])
    for i, hop in enumerate(reversed(received)):
        # Extract IP from "from X (X [IP])"
        ip_match = re.search(r'\[(\d+\.\d+\.\d+\.\d+)\]', hop)
        ip = ip_match.group(1) if ip_match else 'unknown'
        print(f"Hop {i+1}: {ip}")
        print(f"  {hop[:120].strip()}")

    # Red flags
    print("\n=== RED FLAGS ===")
    from_domain = re.search(r'@([^>]+)', msg.get('From', ''))
    rt_domain = re.search(r'@([^>]+)', msg.get('Return-Path', ''))
    if from_domain and rt_domain:
        fd = from_domain.group(1).rstrip('>')
        rd = rt_domain.group(1).rstrip('>')
        if fd.lower() != rd.lower():
            print(f"! From domain ({fd}) != Return-Path domain ({rd})")
    if msg.get('Reply-To'):
        print(f"! Reply-To is set: {msg.get('Reply-To')}")
    if 'dkim=fail' in auth.lower() or 'dkim=none' in auth.lower():
        print("! DKIM failed or missing")
    if 'dmarc=fail' in auth.lower():
        print("! DMARC failed")

if __name__ == '__main__':
    analyze(sys.argv[1])
```

```bash
# Save a suspicious email as .eml and analyze
python3 analyze-email-headers.py suspicious.eml
```

## Quick Reference: Red Flags in Headers

| Flag | What it means |
|------|--------------|
| Reply-To differs from From | Responses will go to attacker |
| Return-Path domain != From domain | Actual sender is different |
| SPF=fail | IP not authorized by sending domain |
| DKIM=fail or missing | Message body may be altered or domain is spoofed |
| DMARC=fail | Both SPF and DKIM failed alignment |
| Received chain starts from unknown IP | Did not originate from claimed domain |
| Multiple X-Originating-IP hops through unfamiliar countries | Routing through proxy/VPS |

## Advanced Email Forensics

### Full Header Analysis for Complex Spoofing

Some sophisticated phishing attempts spoof multiple headers. Here's a forensic approach:

```bash
#!/bin/bash
# Advanced email header forensics

email_file="suspicious.eml"

echo "=== Email Header Forensics ==="
echo ""

# Extract all Received headers
echo "--- Received Chain (bottom to top = originating first) ---"
grep "^Received:" "$email_file" | tac | nl

echo ""
echo "--- Authentication Results ---"
grep -E "^(SPF|DKIM|DMARC|Authentication-Results):" "$email_file"

echo ""
echo "--- Sender Information ---"
grep -E "^(From|Reply-To|Return-Path|Sender):" "$email_file"

echo ""
echo "--- X-Headers (Debugging Info) ---"
grep "^X-" "$email_file"

echo ""
echo "--- IP Address Reverse Lookup ---"
# Extract IPs from Received headers and perform reverse DNS
grep "Received:" "$email_file" | grep -oE "\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\]" | \
    sed 's/[\[\]]//g' | while read ip; do
    echo "IP: $ip"
    dig +short -x "$ip"
done
```

### Building a Phishing Email Database

Track patterns across multiple phishing attempts:

```python
#!/usr/bin/env python3
"""Build searchable database of phishing attempts."""

import sqlite3
import email
import hashlib
from datetime import datetime

class PhishingDatabase:
    def __init__(self, db_file="phishing.db"):
        self.conn = sqlite3.connect(db_file)
        self.init_schema()

    def init_schema(self):
        """Create database tables."""
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS phishing_emails (
                id INTEGER PRIMARY KEY,
                timestamp DATETIME,
                from_domain TEXT,
                claimed_sender TEXT,
                real_sender TEXT,
                subject TEXT,
                sender_ip TEXT,
                sender_country TEXT,
                spf_result TEXT,
                dkim_result TEXT,
                dmarc_result TEXT,
                phishing_type TEXT,
                content_hash TEXT UNIQUE
            )
        """)

        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS indicators (
                email_id INTEGER,
                indicator_type TEXT,
                indicator_value TEXT,
                severity TEXT,
                FOREIGN KEY(email_id) REFERENCES phishing_emails(id)
            )
        """)
        self.conn.commit()

    def analyze_email(self, filepath: str) -> dict:
        """Parse email and extract phishing indicators."""
        with open(filepath, 'rb') as f:
            msg = email.message_from_bytes(f.read())

        analysis = {
            "from_domain": msg.get('From', '').split('@')[-1].rstrip('>'),
            "claimed_sender": msg.get('From', ''),
            "real_sender": msg.get('Return-Path', '').split('@')[-1].rstrip('>'),
            "subject": msg.get('Subject', ''),
            "spf": msg.get('Received-SPF', 'none'),
            "dkim": msg.get('DKIM-Signature', 'none'),
            "dmarc": msg.get('Authentication-Results', 'none'),
            "content_hash": hashlib.sha256(str(msg).encode()).hexdigest()
        }

        # Identify indicators
        indicators = []
        if analysis["from_domain"] != analysis["real_sender"]:
            indicators.append({
                "type": "Domain Mismatch",
                "value": f"{analysis['from_domain']} != {analysis['real_sender']}",
                "severity": "HIGH"
            })

        if "fail" in analysis["spf"].lower():
            indicators.append({
                "type": "SPF Failure",
                "value": analysis["spf"],
                "severity": "HIGH"
            })

        return {
            "analysis": analysis,
            "indicators": indicators
        }

    def record_email(self, filepath: str, phishing_type: str = "unknown"):
        """Record analyzed email in database."""
        result = self.analyze_email(filepath)
        analysis = result["analysis"]

        try:
            cursor = self.conn.execute("""
                INSERT INTO phishing_emails
                (timestamp, from_domain, claimed_sender, real_sender, subject,
                 spf_result, dkim_result, dmarc_result, phishing_type, content_hash)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, (
                datetime.now(),
                analysis["from_domain"],
                analysis["claimed_sender"],
                analysis["real_sender"],
                analysis["subject"],
                analysis["spf"],
                analysis["dkim"],
                analysis["dmarc"],
                phishing_type,
                analysis["content_hash"]
            ))

            email_id = cursor.lastrowid

            # Record indicators
            for indicator in result["indicators"]:
                self.conn.execute("""
                    INSERT INTO indicators
                    (email_id, indicator_type, indicator_value, severity)
                    VALUES (?, ?, ?, ?)
                """, (
                    email_id,
                    indicator["type"],
                    indicator["value"],
                    indicator["severity"]
                ))

            self.conn.commit()
            return email_id
        except sqlite3.IntegrityError:
            print("Email already in database (duplicate)")
            return None

# Usage
db = PhishingDatabase()
email_id = db.record_email("suspicious.eml", "spoofing")
print(f"Recorded email ID: {email_id}")
```

## Integration with Email Clients

### Thunderbird Email Analysis Setup

Create a filter to automatically extract headers:

```javascript
/* Thunderbird addon to auto-extract headers */

mailListener.onStartHeaders = function() {
    // Get current message
    const message = GetSelectedMessage();

    if (message) {
        // Extract headers
        const messenger = Components.classes["@mozilla.org/messenger;1"]
            .createInstance(Components.interfaces.nsIMessenger);

        // Display in console
        console.log("From: " + message.author);
        console.log("Subject: " + message.subject);
        console.log("Date: " + message.date);
    }
};
```

### Gmail Label-Based Automation

Create a label filter to mark suspicious emails:

```javascript
// Gmail script to flag emails with authentication failures

function checkAuthenticationResults() {
    const threads = GmailApp.search('has:nouserlabels');

    threads.forEach(thread => {
        const messages = thread.getMessages();
        messages.forEach(msg => {
            const headers = msg.getHeaders();
            const authResults = headers['Authentication-Results'] || '';

            if (authResults.includes('dkim=fail') || authResults.includes('dmarc=fail')) {
                thread.addLabel(GmailApp.getUserLabelByName('Auth-Failed'));
                thread.markAsSpam();
            }
        });
    });
}

// Schedule to run hourly
ScriptApp.newTrigger('checkAuthenticationResults')
    .timeBased()
    .everyHours(1)
    .create();
```

## Industry-Specific Phishing Patterns

### Banking Phishing Common Headers

| Spoofed Bank | Common Fake Domain | Real Domain Pattern | Detection Signal |
|---|---|---|---|
| Chase Bank | chase-secure.com | chase.com | Domain mismatch + SPF fail |
| Wells Fargo | secure-wellsfargo.net | wellsfargo.com | Reply-To differs |
| PayPal | paypal-verify.com | paypal.com | DKIM fail |
| Amazon | amazon-verify.net | amazon.com | Return-Path mismatch |

### Government Impersonation Patterns

IRS and tax-related phishing commonly spoof with:

```
From: "IRS" <irs@irs-verify-account.com>
Return-Path: <noreply@government-tax-system.net>
Reply-To: <tax-support@verify-identity.com>
```

**Red flags:**
- Real IRS uses irs.gov only
- Will never email without first mailing notice
- Always uses official .gov domains

## Related Reading

- [What to Do If You Click a Phishing Link](/privacy-tools-guide/what-happens-if-you-click-a-phishing-link-on-chrome-steps/)
- [Password Manager Phishing Protection Compared](/privacy-tools-guide/password-manager-phishing-protection-compared/)
- [How to Protect Yourself from QR Code Phishing](/privacy-tools-guide/how-to-protect-yourself-from-qr-code-phishing-quishing-attac/)
- [How to Detect ARP Spoofing Attacks](/privacy-tools-guide/detect-arp-spoofing-attacks-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
- [How to Audit What Source Code AI Coding Tools Transmit](https://theluckystrike.github.io/ai-tools-compared/how-to-audit-what-source-code-ai-coding-tools-transmit-externally/)

---

## Related Articles

- [Email Header Analysis What Metadata Reveals About Your](/privacy-tools-guide/email-header-analysis-what-metadata-reveals-about-your-locat/)
- [Email Security Headers Dmarc Dkim Spf Setup To Prevent](/privacy-tools-guide/email-security-headers-dmarc-dkim-spf-setup-to-prevent-spoofing/)
- [How to Block Tracking Pixels in Email Clients: Setup Guide](/privacy-tools-guide/how-to-block-tracking-pixels-in-email-clients-setup-guide/)
- [How To Check If Your Email Is Being Forwarded](/privacy-tools-guide/how-to-check-if-your-email-is-being-forwarded-without-knowle/)
- [Best Privacy-Focused Email Aliases Service Comparison 2026](/privacy-tools-guide/best-privacy-focused-email-aliases-service-comparison-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
