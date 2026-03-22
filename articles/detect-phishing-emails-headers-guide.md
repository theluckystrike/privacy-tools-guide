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

## Key Takeaways

- **Most email clients hide them by default**: but a 30-second header analysis can confirm whether a suspicious message is legitimate or a spoof.
- **This guide covers how**: to access email headers, the key fields to check, 1. from vs. return-path vs. reply-to, with specific setup instructions
- **Setup and configuration**: Step-by-step instructions included for each tool discussed
- **Practical recommendations**: Specific use-case guidance based on team size and requirements

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

## Related Reading

- [What to Do If You Click a Phishing Link](/privacy-tools-guide/what-happens-if-you-click-a-phishing-link-on-chrome-steps/)
- [Password Manager Phishing Protection Compared](/privacy-tools-guide/password-manager-phishing-protection-compared/)
- [How to Protect Yourself from QR Code Phishing](/privacy-tools-guide/how-to-protect-yourself-from-qr-code-phishing-quishing-attac/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
