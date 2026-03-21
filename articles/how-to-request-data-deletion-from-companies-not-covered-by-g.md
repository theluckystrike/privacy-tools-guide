---
layout: default
title: "How To Request Data Deletion From Companies Not Covered By G"
description: "Learn practical methods to request data deletion from companies that fall outside GDPR and CCPA protections. Includes email templates, legal"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-request-data-deletion-from-companies-not-covered-by-g/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
---

{% raw %}

Send formal deletion requests explicitly referencing applicable state laws (Virginia VCDPA, Colorado CPA, Connecticut CTDPA, Utah UCPA, Texas TDPSA) even to non-GDPR/CCPA companies, as many voluntarily comply. If companies refuse, escalate to state attorneys general, the FTC, or BBB. For companies beyond legal reach, minimize future data sharing through temporary emails, pseudonyms, and privacy tools, or pursue small claims court for significant violations.

## Understanding the Data Protection Gap

When GDPR and CCPA don't apply to a company, your personal data rights become significantly limited. GDPR covers companies with EU connections—either offering goods/services to EU residents or monitoring their behavior. CCPA applies primarily to for-profit businesses meeting specific thresholds (>$25M revenue, data >100K consumers, or deriving 50%+ revenue from selling personal data).

This leaves a massive gap. Many US-based companies, startups, small businesses, and organizations operating outside these jurisdictions collect and store your personal information without legal obligation to delete it when you request.

For developers and power users, this means building automation around data deletion requests requires understanding both legal frameworks and practical API-driven approaches.

## What Data Companies Typically Hold

Before requesting deletion, understand what companies might have:

- **Account information**: Email, username, password (hashed)
- **Usage data**: Activity logs, feature usage patterns
- **Communication data**: Support tickets, chat logs, emails
- **Payment information**: Billing addresses, partial payment data
- **Device and technical data**: IP addresses, browser fingerprints
- **Third-party data**: Information from data brokers or partners

## Automating Data Discovery with Developer Tools

Power users can script discovery of where their data lives:

```bash
# Search for account data across known services
#!/bin/bash
# Script to check known data broker registrations

SERVICES=("haveibeenpwned" "dehashed" "breachdirectory" "passwordscanner")

for service in "${SERVICES[@]}"; do
    echo "Checking $service for your email..."
    curl -s "https://$service.org/api/your@email.com" | jq '.'
done
```

## Step-by-Step Method to Request Data Deletion

### Step 1: Identify the Company and Their Privacy Practices

First, locate the company's privacy policy. Search for terms like:
- "data deletion"
- "delete my account"
- "remove my data"
- "privacy policy"

Check if they have:
- A self-service account deletion option
- A designated privacy contact email
- A formal data deletion request process

### Step 2: Draft Your Data Deletion Request

Create a clear, formal request. Include:

```text
Subject: Data Deletion Request - [Your Account Email/ID]

To the Privacy Team at [Company Name],

I am requesting the deletion of all personal data your company holds about me.

Account identifier: [Your email or username]
Registered name: [Your full name if known]
Account created: [Approximate date if known]

This request is made under [applicable law if any, e.g., California Civil Code § 1798.105, or as a general privacy request].

Please confirm receipt of this request and provide timeline for deletion.

Sincerely,
[Your Name]
[Your Email]
[Your Address]
```

### Step 3: Send the Request Through Multiple Channels

- **Email**: Send to privacy@company.com, support@company.com, or the address listed in their privacy policy
- **Contact form**: Many companies have web forms—use these as they create documentation
- **Certified mail**: For critical requests, postal mail creates legal weight
- **Social media**: Sometimes effective for companies with active social presence

### Step 4: Follow Up

- Wait 10-14 business days for initial response
- Send follow-up if no response
- Document all communications

## Alternative Strategies

### State-Level Privacy Laws

Beyond CCPA, several US states have enacted privacy laws:
- **Virginia (VCDPA)**: Consumer rights including deletion
- **Colorado (CPA)**: Right to delete
- **Connecticut (CTDPA)**: Deletion rights
- **Utah (UCPA)**: Consumer rights
- **Texas (TDPSA)**: Rights and opt-outs

Check if the company operates in these states—they may still honor deletion requests.

### Industry-Specific Regulations

Certain industries have their own data protection rules:
- **HIPAA**: Healthcare providers and their business associates
- **FERPA**: Educational institutions
- **GLBA**: Financial institutions
- **COPPA**: Companies collecting data from children under 13

### Programmatic Deletion Requests

For developers building privacy tools, here's a Python template for managing deletion requests:

```python
import smtplib
from email.mime.text import MIMEText
import json
from datetime import datetime

class DataDeletionRequest:
    def __init__(self, email, company_config):
        self.email = email
        self.company = company_config['name']
        self.contact = company_config['contact']
        self.sent_date = None
        
    def generate_request_email(self, subject, body_template):
        msg = MIMEText(body_template.format(email=self.email))
        msg['Subject'] = subject
        msg['From'] = 'your-email@example.com'
        msg['To'] = self.contact
        return msg
    
    def send_via_smtp(self, smtp_config):
        with smtplib.SMTP(smtp_config['host'], smtp_config['port']) as server:
            server.starttls()
            server.login(smtp_config['user'], smtp_config['pass'])
            # Send request
            self.sent_date = datetime.now()
            
# Usage
companies = [
    {'name': 'Company A', 'contact': 'privacy@companya.com'},
    {'name': 'Company B', 'contact': 'dpo@companyb.com'},
]

for company in companies:
    request = DataDeletionRequest('your@email.com', company)
    # Send deletion request
```

### Direct Technical Methods

Where legal requests fail, technical options exist:

**Minimize data shared**:
- Use temporary email services for sign-ups
- Use pseudonyms where permitted
- Avoid connecting real identity to accounts

**Use privacy tools**:
- Browser extensions that block tracking
- VPN services to mask IP addresses
- Cookie consent managers to limit data collection

### Data Breach Use

If the company has experienced a data breach, use this as use. Under many state laws, companies must maintain specific security practices. A breach history may encourage compliance.

## What to Do If They Refuse

### Document Everything

Keep records of:
- All requests sent
- All responses received
- Dates and methods of communication
- Any promises made by the company

### File Complaints

- **State Attorney General**: Many states have consumer protection divisions
- **FTC**: Report deceptive privacy practices at ftc.gov/complaint
- **Better Business Bureau**: File complaints affecting business practices

### Consider Legal Action

Small claims court is an option for significant data mishandling:
- Filing fees are typically low ($50-100)
- No attorney required
- Companies often settle to avoid court

## Template Collection

### Simple Deletion Request Email

```email
Subject: Account Deletion Request

Hello,

Please delete my account and all associated personal data.

Email: [your@email.com]
Username: [your-username]

Regards,
[Your Name]
```

### Formal Legal-Style Request

```email
Subject: Formal Data Deletion Request Under [State Law]

To the Data Protection Officer:

This letter constitutes a formal request to delete all personal information you maintain about me.

Identifying Information:
- Email: [email]
- Account ID: [if known]
- Name: [name]
- Address: [address]

I request that you:
1. Delete all personal data within 30 days
2. Confirm deletion in writing
3. Notify any third parties with whom you've shared my data

If you cannot fulfill this request, please provide specific legal basis for refusal.

[Your signature]
[Date]
```

### GDPR-Style Request (Even for Non-Covered Companies)

```email
Subject: Data Subject Access Request and Deletion Request

To Whom It May Concern:

Regardless of whether your organization falls under GDPR jurisdiction, I formally request:

1. Access to all personal data you hold about me
2. Correction of any inaccurate data
3. Deletion of all personal data (right to erasure)
4. Portability of my data in machine-readable format

I reserve all rights under applicable privacy laws and expect good-faith compliance.

[Your details]
```



## Related Articles

- [How To Request Algorithmic Transparency From Companies Makin](/privacy-tools-guide/how-to-request-algorithmic-transparency-from-companies-makin/)
- [Ledger Data Breach Lessons How Hardware Wallet Companies Can](/privacy-tools-guide/ledger-data-breach-lessons-how-hardware-wallet-companies-can/)
- [GDPR Data Subject Access Request Template](/privacy-tools-guide/gdpr-data-subject-access-request-template/)
- [Set Up Data Subject Access Request Workflow](/privacy-tools-guide/how-to-set-up-data-subject-access-request-workflow-for-gdpr-/)
- [Using curl for LinkedIn API](/privacy-tools-guide/social-media-data-request-download-guide-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
