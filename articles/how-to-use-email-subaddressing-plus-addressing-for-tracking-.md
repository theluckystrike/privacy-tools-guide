---
layout: default
title: "Use Email Subaddressing Plus Addressing For Tracking Which"
description: "A practical guide to using email subaddressing and addressing techniques to track which services leak your email address. Perfect for developers"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-use-email-subaddressing-plus-addressing-for-tracking-which-services-leak-your-address/
categories: [guides, security]
tags: [privacy-tools-guide]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
---

{% raw %}

Email subaddressing (also known as plus addressing or tagged addressing) is a powerful technique that lets you create infinite email variations using a single inbox. When combined with strategic addressing patterns, you can not only organize your email but also detect which services leak or sell your personal information. This guide covers practical implementations for developers and power users who want to maintain control over their inbox and track data handling practices.

Cloudflare Email Routing (free) setup example
1.
- This RFC 5233 compliant: feature works with Gmail, Outlook, iCloud, Proton Mail, and most modern email services.
- How do I get: started quickly? Pick one tool from the options discussed and sign up for a free trial.
- What is the learning: curve like? Most tools discussed here can be used productively within a few hours.
- Choose your base email: - use a privacy-respecting provider 2.

Table of Contents

- [Prerequisites](#prerequisites)
- [Advanced - Custom Domain Subaddressing](#advanced-custom-domain-subaddressing)
- [Advanced - Automated Leak Detection System](#advanced-automated-leak-detection-system)
- [Service-Level Comparison - Which Services Leak Most](#service-level-comparison-which-services-leak-most)
- [Troubleshooting](#troubleshooting)

Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


Step 1 - Understand Email Subaddressing

Most email providers support a simple syntax: anything after a plus sign (`+`) in your email address gets ignored during delivery but gets preserved in the recipient metadata. If your primary email is `developer@example.com`, you can use `developer+github@example.com` for GitHub registrations. Both addresses deliver to the same inbox, but the recipient sees the full address.

This RFC 5233 compliant feature works with Gmail, Outlook, iCloud, Proton Mail, and most modern email services. The key advantage is that you can create unique addresses for every service without managing multiple accounts.

Step 2 - Set Up Subaddressing for Tracking

Create a systematic approach to tracking email usage across services. Generate aliases using a consistent format that identifies the service name:

```javascript
// Generate subaddresses for service tracking
function createServiceEmail(serviceName, baseEmail = 'developer@example.com') {
  const [local, domain] = baseEmail.split('@');
  return `${local}+${serviceName.toLowerCase().replace(/\s+/g, '-')}@${domain}`;
}

// Usage examples
const serviceEmails = {
  github: createServiceEmail('github'),
  aws: createServiceEmail('aws'),
  linkedin: createServiceEmail('linkedin'),
  newsletter: createServiceEmail('newsletter-weekly'),
  ecommerce: createServiceEmail('shop-amazon')
};

console.log(serviceEmails.github);   // developer+github@example.com
console.log(serviceEmails.aws);      // developer+aws@example.com
```

This script generates predictable aliases that you can document in a password manager or dedicated tracking system. When you sign up for a new service, create a subaddress that clearly identifies the service name.

Step 3 - Detecting Address Leaks

Once you've been using subaddresses for a while, detecting leaks becomes straightforward. Create a dedicated email filter that labels incoming mail by the alias used:

```bash
Gmail filter example - create filters for each service
Match - to:(developer+github@example.com)
Action - Add label "service/github", never mark as spam
```

When `developer+github@example.com` starts receiving spam from unrelated domains, you know GitHub either experienced a breach or shared your data with third parties. Similarly, if your `linkedin+newsletter@example.com` address suddenly receives promotional emails about insurance, LinkedIn's data sharing practices are now affecting your inbox.

Advanced - Custom Domain Subaddressing

For maximum control, pair subaddressing with a custom domain. This approach provides additional benefits:

1. Portability: Change email providers without updating addresses
2. Flexibility: Create unlimited aliases without provider limits
3. Branding: Use professional domain names

```bash
Cloudflare Email Routing (free) setup example
1. Add your domain to Cloudflare
2. Enable Email Routing
3. Configure catch-all to forward to your primary inbox
4. Start using aliases like:
   - dev@yourdomain.com
   - dev+shop@yourdomain.com
   - dev+travel@yourdomain.com
```

Services like Cloudflare Email Routing, Forward Email, and ImprovMX offer free catch-all forwarding. Combined with a privacy-respecting inbox provider, you gain full control over incoming mail.

Step 4 - Identifying Service Data Breaches

When a service you registered with experiences a breach, attackers often obtain email addresses. If you're using subaddresses, you can identify the compromised service immediately:

1. Monitor incoming mail to subaddresses for spam patterns
2. Track unsubscribe behavior - legitimate services honor unsubscribe requests
3. Note reply patterns - some services send automated replies to verify email validity

Create a simple logging system:

```javascript
// Track which addresses receive mail
const emailLog = [];

function logIncomingEmail(fullAddress) {
  const match = fullAddress.match(/^([^@]+)\+([^@]+)@(.+)$/);
  if (match) {
    const [, user, service, domain] = match;
    emailLog.push({
      timestamp: new Date().toISOString(),
      service: service,
      address: fullAddress
    });
    console.log(`Email received for service: ${service}`);
  }
}

// Later: analyze emailLog to detect anomalies
function detectLeak(serviceName) {
  const relevantLogs = emailLog.filter(e => e.service === serviceName);
  const uniqueSenders = new Set(relevantLogs.map(e => extractDomain(e.from)));
  console.log(`${serviceName} has been contacted by ${uniqueSenders.size} unique senders`);
}
```

Step 5 - Practical Implementation Strategy

Start with these steps to build your tracking system:

1. Choose your base email - use a privacy-respecting provider
2. Adopt a naming convention - `service-purpose-date` format works well
3. Create a tracking document - maintain a list of all active aliases
4. Set up filters - automatically label incoming mail by service
5. Review quarterly - check for unexpected senders or patterns

When you notice a leak, you have options:
- Block the specific alias entirely
- Contact the service to request data removal
- Use the alias for targeted spam filtering in your email client

Step 6 - Common Pitfalls to Avoid

Some services don't accept plus addresses due to poor validation. Workarounds include:

- Using dots in Gmail (`d.e.v.e.l.o.p.e.r@example.com` is identical to `developer@example.com` but appears unique)
- Creating actual aliases in your email provider settings
- Using catch-all domain forwarding for maximum flexibility

Avoid using easily guessable patterns like sequential numbers (`user+1@example.com`, `user+2@example.com`) as these can be trivially stripped by spam systems.

Step 7 - Automate Alias Management

For power users managing many services, automate the process:

```javascript
// Automated alias generator with logging
class EmailAliasManager {
  constructor(baseEmail, storage) {
    this.baseEmail = baseEmail;
    this.aliases = new Map();
    this.storage = storage;
  }

  generate(service, purpose = 'general') {
    const key = `${service}-${purpose}`;
    if (!this.aliases.has(key)) {
      const alias = this.createAlias(service, purpose);
      this.aliases.set(key, alias);
      this.storage.save(key, alias);
    }
    return this.aliases.get(key);
  }

  createAlias(service, purpose) {
    const [local, domain] = this.baseEmail.split('@');
    const timestamp = Date.now().toString(36);
    return `${local}+${service}-${purpose}-${timestamp}@${domain}`;
  }

  lookup(alias) {
    return [...this.aliases.entries()].find(([_, a]) => a === alias);
  }
}
```

This approach creates discoverable aliases that include service name, purpose, and a timestamp for uniqueness while remaining readable.

Step 8 - Domain-Based Aliasing for Professional Separation

For developers and power users managing separate professional identities, custom domain aliases provide stronger separation than plus addressing:

```bash
Set up Mailgun (or equivalent) for email routing
curl -X POST https://api.mailgun.net/v3/mg.yourdomain.com/routes \
  -F priority=0 \
  -F pattern='github-*@yourdomain.com' \
  -F action='forward("github@forwarding.address")' \
  -F action='stop()' \
  -u api:YOUR_API_KEY
```

This approach scales better than plus addressing and provides professional aliases for different personas (dev@yourdomain.com, security@yourdomain.com, etc.).

Step 9 - Integrate with Password Managers for Tracking

Store subaddresses in your password manager with metadata tags to correlate with leak detection:

```json
{
  "id": "github_2026_01",
  "name": "GitHub Account",
  "login": {
    "email": "developer+github@example.com",
    "username": "dev-github"
  },
  "notes": "Service: GitHub, Purpose: Development",
  "tags": ["service-tracking", "github", "dev-work"],
  "fields": {
    "service_domain": "github.com",
    "service_type": "code-repository",
    "created_date": "2026-01-15"
  }
}
```

When you receive unexpected mail to `developer+github@example.com`, look up the entry in your password manager to immediately identify the source.

Advanced - Automated Leak Detection System

Build a tracking system that monitors aliases in real-time:

```python
#!/usr/bin/env python3
import imaplib
import json
from datetime import datetime, timedelta
from collections import defaultdict

class AliasLeakDetector:
    def __init__(self, email, password, imap_server='imap.gmail.com'):
        self.email = email
        self.imap = imaplib.IMAP4_SSL(imap_server)
        self.imap.login(email, password)
        self.leak_log = defaultdict(list)

    def get_emails_since(self, minutes=60):
        """Fetch emails from the last N minutes."""
        self.imap.select('INBOX')

        # Search for recent emails
        since_date = (datetime.now() - timedelta(minutes=minutes)).strftime('%d-%b-%Y')
        status, messages = self.imap.search(None, f'SINCE {since_date}')

        email_ids = messages[0].split()
        return email_ids

    def parse_email(self, email_id):
        """Extract sender and recipient alias."""
        status, data = self.imap.fetch(email_id, '(RFC822)')

        import email as email_lib
        msg = email_lib.message_from_bytes(data[0][1])

        to_addr = msg['To']
        from_addr = msg['From']

        # Extract alias from recipient address
        if '+' in to_addr:
            alias = to_addr.split('+')[1].split('@')[0]
            return {
                'alias': alias,
                'from': from_addr,
                'to': to_addr,
                'date': msg['Date']
            }
        return None

    def detect_leaks(self):
        """Identify unexpected senders for each alias."""
        email_ids = self.get_emails_since()

        for email_id in email_ids:
            parsed = self.parse_email(email_id)
            if parsed:
                alias = parsed['alias']
                from_domain = parsed['from'].split('@')[-1].rstrip('>')

                self.leak_log[alias].append({
                    'from': parsed['from'],
                    'domain': from_domain,
                    'date': parsed['date']
                })

    def report_leaks(self, expected_domains_file='expected_domains.json'):
        """Compare against expected senders."""
        with open(expected_domains_file, 'r') as f:
            expected = json.load(f)

        for alias, senders in self.leak_log.items():
            unique_domains = set(s['domain'] for s in senders)
            allowed_domains = set(expected.get(alias, []))

            unexpected = unique_domains - allowed_domains
            if unexpected:
                print(f"[LEAK] {alias}: unexpected sender(s): {unexpected}")
            else:
                print(f"[OK] {alias}: all senders are expected")

Usage
detector = AliasLeakDetector('your@email.com', 'password')
detector.detect_leaks()
detector.report_leaks()
```

Service-Level Comparison - Which Services Leak Most

Track which services expose your address to unexpected senders:

```python
import json
from collections import defaultdict

Log structure
leak_report = {
    "github": {
        "registration_date": "2026-01-15",
        "expected_senders": ["noreply@github.com", "support@github.com"],
        "unexpected_senders": [
            "marketing@github-partner.com",
            "survey@external-analytics.com"
        ],
        "leak_confirmed": True
    },
    "amazon": {
        "registration_date": "2026-01-20",
        "expected_senders": ["order-confirmation@amazon.com", "ads@amazon.com"],
        "unexpected_senders": [],
        "leak_confirmed": False
    }
}

Calculate leak rates
total_services = len(leak_report)
leaked_services = sum(1 for s in leak_report.values() if s['leak_confirmed'])
leak_rate = (leaked_services / total_services) * 100

print(f"Leak rate: {leak_rate}% of services ({leaked_services}/{total_services})")
```

Step 10 - Gmail-Specific Advanced Filters

Gmail's powerful filter system enables sophisticated leak detection without third-party tools:

1. Create labels for each service: `services/github`, `services/aws`, etc.
2. Create filters with these expressions:

```
from:(developer+github@example.com) OR to:(developer+github@example.com)
```

3. Assign multiple labels:
 - `services/github` (primary)
 - `services/github-unexpected` (for unexpected senders)

4. Use label hierarchy to organize:

```
services/
   github/
      expected
      unexpected
   aws/
      expected
      unexpected
```

Step 11 - Preventing Cross-Service Correlation

When aliases are leaked and sold to data brokers, multiple services can correlate your activity. Mitigate this by:

1. Using unique patterns per service:
 ```
   github: developer+gh-abc123@example.com
   aws: developer+aws-xyz789@example.com
   linkedin: developer+li-def456@example.com
   ```

2. Rotating aliases periodically: Every 90 days, create new aliases and migrate accounts
3. Monitoring for pattern-based tracking: Check if multiple leaked emails follow a predictable sequence

Troubleshooting

Configuration changes not taking effect

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

Permission denied errors

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

Connection or network-related failures

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


Frequently Asked Questions

Who is this article written for?

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

How current is the information in this article?

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

Are there free alternatives available?

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

How do I get started quickly?

Pick one tool from the options discussed and sign up for a free trial. Spend 30 minutes on a real task from your daily work rather than running through tutorials. Real usage reveals fit faster than feature comparisons.

What is the learning curve like?

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

Related Articles

- [Email Tracking Pixel Detection](/email-tracking-pixel-detection-how-to-identify-and-block-spy/)
- [How to Block Tracking Pixels in Email Clients: Setup Guide](/how-to-block-tracking-pixels-in-email-clients-setup-guide/)
- [Set Up Mail In A Box Private Email Server Complete 2026](/how-to-set-up-mail-in-a-box-private-email-server-complete-2026-guide/)
- [Set Up Catch All Email Domain For Unlimited Private](/how-to-set-up-catch-all-email-domain-for-unlimited-private-aliases/)
- [Someone Signed Up for Services Using My Email](/someone-signed-up-for-services-using-my-email-what-to-do/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)
```
```
{% endraw %}
