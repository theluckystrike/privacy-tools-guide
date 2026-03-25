---
layout: default
title: "How to Detect if Your Email Is Compromised"
description: "Practical steps to find out if your email account has been breached. Check breach databases, audit login activity, and secure your account fast."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: detect-email-compromise-guide
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

A compromised email account is a skeleton key. Attackers use it to reset passwords on every other account, read private correspondence, and impersonate you. Most breaches go unnoticed for weeks or months.

This guide gives you concrete methods to detect compromise, investigate the extent of damage, and lock things down.

Table of Contents

- [Signs Your Email May Be Compromised](#signs-your-email-may-be-compromised)
- [Step 1 - Check Breach Databases](#step-1-check-breach-databases)
- [Step 2 - Review Active Sessions](#step-2-review-active-sessions)
- [Step 3 - Check for Email Forwarding Rules](#step-3-check-for-email-forwarding-rules)
- [Step 4 - Check Connected Apps and OAuth Grants](#step-4-check-connected-apps-and-oauth-grants)
- [Step 5 - Examine Recent Account Activity](#step-5-examine-recent-account-activity)
- [Step 6 - Check for Credential Leaks on Paste Sites](#step-6-check-for-credential-leaks-on-paste-sites)
- [Step 7 - Secure the Account](#step-7-secure-the-account)
- [Ongoing Monitoring](#ongoing-monitoring)

Signs Your Email May Be Compromised

- Contacts report receiving spam or phishing messages from you
- You receive password reset emails you didn't request
- Unknown devices appear in your account's active session list
- Emails are marked as read that you haven't opened
- Your sent folder has messages you didn't write
- You get locked out of your own account
- Emails are being forwarded to an address you don't recognize

Any single one of these warrants immediate investigation.

Step 1 - Check Breach Databases

Start with publicly known breaches. Your credentials may have been leaked from a third-party service you used the same password on.

Have I Been Pwned

```bash
Query via API (no account needed)
curl -s "https://haveibeenpwned.com/api/v3/breachedaccount/your@email.com" \
  -H "hibp-api-key: YOUR_API_KEY" | python3 -m json.tool
```

Or visit `https://haveibeenpwned.com` directly. Enter your email and check every listed breach.

For each breach, note:
- What data was exposed (passwords, phone numbers, addresses)
- When the breach occurred
- Whether you still use the same password

Check Your Password Directly

HIBP also lets you check if a specific password appears in breach data without sending the password itself (uses k-anonymity):

```bash
Hash your password with SHA-1, send only the first 5 chars
echo -n "yourpassword" | sha1sum | tr '[:lower:]' '[:upper:]'
Take first 5 chars of output, e.g. "2B3A8"

curl -s "https://api.pwnedpasswords.com/range/2B3A8"
Returns hash suffixes and how many times they appeared in breaches
```

If your full SHA-1 hash suffix appears in the results with a non-zero count, change that password immediately.

Step 2 - Review Active Sessions

Every major email provider lets you view active sessions. Check these first.

Gmail

1. Scroll to the bottom of the Gmail inbox
2. Click "Details" next to "Last account activity"
3. A popup shows all recent sessions with IP addresses, device types, and timestamps

Look for:
- Unrecognized countries or cities
- Sessions at odd hours
- Device types you don't own

If anything looks wrong, click "Sign out all other web sessions."

Proton Mail

Settings > Security > Active sessions. lists all devices with last activity timestamps.

Outlook / Microsoft

Visit `https://account.microsoft.com/security` and check "Review recent activity." Microsoft shows login locations on a map.

Check via CLI for Self-Hosted Mail

If you run your own mail server, check auth logs:

```bash
Postfix/Dovecot auth attempts
sudo grep "authentication failed\|login" /var/log/mail.log | tail -100

Failed IMAP logins
sudo grep "imap-login" /var/log/dovecot.log | grep "Aborted login\|Disconnected" | tail -50
```

Step 3 - Check for Email Forwarding Rules

Attackers frequently set up silent forwarding rules so they keep receiving your email even after you change your password.

Gmail

Settings > "See all settings" > Forwarding and POP/IMAP tab

- Forwarding: should say "Forwarding is disabled" unless you set it up
- Check the forwarding address if enabled

Also check Filters and Blocked Addresses. attackers sometimes create filters that auto-forward specific messages or delete security alerts.

Outlook

Settings > Mail > Forwarding. should be disabled.

Settings > Mail > Rules. look for any rules you didn't create, especially ones that forward, delete, or move emails.

Check via Gmail API

```python
import googleapiclient.discovery
from google.oauth2.credentials import Credentials

Requires Google API credentials setup
creds = Credentials.from_authorized_user_file('token.json')
service = googleapiclient.discovery.build('gmail', 'v1', credentials=creds)

List forwarding addresses
result = service.users().settings().forwardingAddresses().list(userId='me').execute()
print(result)

List filters
filters = service.users().settings().filters().list(userId='me').execute()
for f in filters.get('filter', []):
    print(f)
```

Step 4 - Check Connected Apps and OAuth Grants

A compromised account may have authorized malicious third-party apps that maintain access even after you change your password.

Gmail OAuth Apps

Visit `https://myaccount.google.com/permissions`

Revoke anything you don't recognize or no longer use.

Microsoft

Visit `https://account.microsoft.com/privacy/app-access`

General approach

For any suspicious OAuth app:
1. Revoke the app's access immediately
2. Note what data the app had access to (mail, contacts, calendar)
3. Assume that data has been copied

Step 5 - Examine Recent Account Activity

Gmail Activity Log

Gmail's full activity log is at `https://myactivity.google.com`. this shows searches, reads, and interactions with your Google account.

Download Your Data

Request a complete export of your account data to check what's there:

- Gmail - `https://takeout.google.com`
- Proton: Settings > Account > Account data export

This gives you a baseline of what an attacker could have accessed.

Check Sent Folder

```bash
If using Thunderbird or similar with local storage, search for unauthorized sent mail
grep -r "From - your@email.com" ~/.thunderbird/*/Mail/*/Sent* | tail -50
```

Step 6 - Check for Credential Leaks on Paste Sites

Credentials sometimes appear on paste sites like Pastebin before they're indexed by HIBP.

Use Google dorking:

```
site:pastebin.com "your@email.com"
site:rentry.co "your@email.com"
```

Also search your username, not just email address.

Step 7 - Secure the Account

Once you've identified a compromise:

1. Change your password immediately. use a password manager to generate a 20+ character random password
2. Enable two-factor authentication. hardware key (YubiKey) or TOTP app, not SMS
3. Sign out all other sessions. invalidates any stolen session tokens
4. Remove unauthorized forwarding rules and filters
5. Revoke unknown OAuth apps
6. Check your recovery email and phone number. attackers change these to lock you out
7. Alert your contacts. let them know if you sent spam
8. Check other accounts. if the same password was used elsewhere, change all of them
9. File a report with your email provider if you suspect targeted attack

Ongoing Monitoring

Set up monitoring to catch future compromise faster:

```bash
HIBP monitoring. requires paid API subscription
Alerts you when your email appears in new breaches
curl -s "https://haveibeenpwned.com/api/v3/subscriptionstatus" \
  -H "hibp-api-key: YOUR_API_KEY"
```

Also enable login notifications in your email provider's security settings. most providers can email or push-notify you on new sign-ins from unfamiliar devices.

Frequently Asked Questions

How long does it take to detect if your email is compromised?

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

- [How To Tell If Your Email Account Was Hacked Right](/how-to-tell-if-your-email-account-was-hacked-right-now/)
- [How To Check If Your Email Is Being Forwarded](/how-to-check-if-your-email-is-being-forwarded-without-knowle/)
- [Best Privacy-Focused Email Alternatives to Gmail 2026](/best-privacy-focused-email-alternatives-to-gmail-2026/)
- [How To Detect If Your Email Address Has Been Sold](/how-to-detect-if-your-email-address-has-been-sold-to-marketi/)
- [Email Account Inheritance Can Executor Legally Access](/email-account-inheritance-can-executor-legally-access-deceas/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)
```
```
{% endraw %}
