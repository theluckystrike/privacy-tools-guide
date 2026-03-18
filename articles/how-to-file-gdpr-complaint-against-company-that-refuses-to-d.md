---
layout: default
title: "How to File a GDPR Complaint Against a Company That."
description: "A technical guide for developers and power users on exercising GDPR data deletion rights, escalating complaints, and using regulatory tools when."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-file-gdpr-complaint-against-company-that-refuses-to-d/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
---

{% raw %}
Filing a GDPR complaint with your national Data Protection Authority (e.g., CNIL in France, ICO in UK, Datenschutzbehörde in Germany) requires documented evidence of your erasure request, the company's refusal response with timestamps, proof the company has legitimate basis to retain data, and detailed description of what personal data remains stored. Most DPAs provide online complaint forms; you upload copies of your deletion request email and their response, cite the specific GDPR article (17 for erasure), and describe the personal data categories. DPAs prioritize cases where companies clearly refuse or ignore requests—formal complaints carry significant regulatory weight because violations trigger fines up to 4% of annual revenue. Documentation matters: timestamps from registered emails, screenshots of account data still visible, and your explanation of why their retention justification fails (contract ended, data unrelated to purpose, or no legal basis) transforms "they won't delete me" into enforceable regulatory actions.

## Understanding Your Rights Under GDPR

Before filing a complaint, ensure your deletion request is valid. GDPR Article 17 applies when:

- The data is no longer necessary for the original purpose
- You withdraw consent (and there's no other legal basis)
- You object to processing
- The data was processed unlawfully
- Deletion is required to comply with legal obligations

Companies can refuse if they must retain data for legal compliance, public interest, or establishing/defending legal claims. However, they must provide **specific** reasoning—not vague boilerplate.

## Step 1: Document Your Original Deletion Request

Always keep records of your initial request. For developers, this means using verifiable methods:

```bash
# Send deletion request via email with read receipt
# Using curl to send via your own SMTP server
curl -s --mail-from <your-email> \
  --mail-rcpt <company-privacy-email> \
  smtp://mail.yourserver.com \
  -H "Subject: GDPR Data Deletion Request" \
  -H "X-Request-ID: $(uuidgen)" \
  --data-urlencode "body@request.txt"
```

Store the request ID and timestamp in a local database or encrypted file:

```python
import sqlite3
import json
from datetime import datetime

def log_deletion_request(company_name, email, request_id):
    conn = sqlite3.connect('gdpr_requests.db')
    conn.execute('''CREATE TABLE IF NOT EXISTS requests
        (id INTEGER PRIMARY KEY, company TEXT, email TEXT,
         request_id TEXT, request_date TEXT, status TEXT)''')
    conn.execute('INSERT INTO requests VALUES (NULL, ?, ?, ?, ?, ?)',
        (company_name, email, request_id, datetime.now().isoformat(), 'pending'))
    conn.commit()
    conn.close()
```

## Step 2: Escalate Through Company Channels

Most companies have a formal escalation process. Check their privacy policy for:

- Designated privacy officer contact
- Supervisory authority reference
- Internal appeal process

Send a formal escalation email referencing GDPR explicitly:

```
Subject: GDPR Article 17 Escalation - Request ID: [YOUR-REFERENCE]

I am escalating my data deletion request ([original date]).
Under GDPR Article 17, you must provide specific reasoning if refusing.
Failure to comply within 30 days will result in a formal complaint
to your supervisory authority.

Regards,
[Your Name]
[Your Email]
[Account ID if applicable]
```

## Step 3: File a Formal Complaint with Your Supervisory Authority

If the company ignores you or refuses without valid justification, file a complaint with your local Data Protection Authority (DPA). Each EU member state has one—find yours on the European Data Protection Board website.

### Required Information for Your Complaint:

1. **Your identity**: Name, email, address
2. **Company details**: Name, website, registered address
3. **Timeline**: Original request date, follow-up dates, response (if any)
4. **Evidence**: Copies of all correspondence, request IDs
5. **Violation**: Why you believe GDPR was violated

### Using Official Complaint Portals

Most DPAs now accept online complaints. Here are direct links to major authorities:

- **UK**: [ICO Complaints](https://ico.org.uk/make-a-complaint/)
- **Germany**: [BfDI Online-Beschwerde](https://www.bfdi.bund.de/)
- **France**: [CNIL Plainte](https://www.cnil.fr/plainte)
- **Ireland**: [DPC Complaints](https://www.dataprotection.ie/complaints)

Some authorities provide APIs for developers:

```javascript
// Example: Submitting complaint via ICO API (if available)
const complaint = {
  yourName: "Your Name",
  yourEmail: "your@email.com",
  companyName: "Company Ltd",
  companyWebsite: "https://company.com",
  incidentDate: "2026-01-15",
  description: "Company refused GDPR deletion request without valid basis",
  evidence: [base64_encoded_email_1, base64_encoded_email_2]
};

fetch('https://api.ico.org.uk/v1/complaints', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer YOUR_API_KEY'
  },
  body: JSON.stringify(complaint)
});
```

## Step 4: Leverage Technical Tools

For developers and power users, several tools can help automate and document the process:

### Browser Extensions

- **Privacy Badge**: Adds timestamps to sent emails
- **HTTP Archive Recorder**: Captures website privacy policy changes

### Command-Line Tools

```bash
# Use whois to find company registration details
whois company-domain.com

# Use curl to check for privacy policy changes
curl -s https://company.com/privacy | grep -i "delete\|erasure\|rights"

# Track email delivery
dig +short TXT company-domain.com | grep "v=spf"
```

### Automated Request Tracking

Build a simple tracking system:

```python
# Track complaint status across multiple authorities
class GDPRComplaint:
    def __init__(self, company, authority, submitted_date):
        self.company = company
        self.authority = authority
        self.submitted_date = submitted_date
        self.status = "submitted"
        self.case_reference = None
    
    def update_status(self, new_status, case_ref=None):
        self.status = new_status
        self.case_reference = case_ref or self.case_reference
        # Notify user via webhook
        webhook_url = os.environ.get('DISCORD_WEBHOOK')
        if webhook_url:
            requests.post(webhook_url, json={
                "content": f"GDPR complaint update: {self.company} - {self.status}"
            })
```

## Step 5: Consider Legal Action

If the DPA doesn't resolve your complaint satisfactorily, you can:

1. **Appeal the DPA decision** within 2-3 months
2. **Direct legal action** against the company (Article 79 GDPR)
3. **Class action** with other affected users

For developers, consider open-source legal tools:

- **GDPR Data Request Templates**: [github.com/lorenzer/gdpr-templates](https://github.com/lorenzer/gdpr-templates)
- **Automated Request Scripts**: [github.com/j marek/gdpr-cli](https://github.com/jmarek/gdpr-cli)

## Common Pitfalls to Avoid

- **Missing deadlines**: Companies have 30 days (extendable to 60 for complex requests)
- **Wrong authority**: File in your country of residence, not the company's location
- **Insufficient evidence**: Always keep timestamped copies of all communication
- **Accepting vague refusals**: Demand specific legal basis under GDPR

## What Happens After You Complain

The DPA will investigate. Typical outcomes include:

- **Warning**: Company must comply within specified timeframe
- **Fine**: Up to €20 million or 4% of global annual turnover
- **Order**: Compulsory deletion with audit requirements
- ** Suspension**: Processing ban until compliance achieved

You receive updates throughout the process and can provide additional evidence if needed.

---

Exercise your GDPR rights systematically. Document everything, escalate formally, and use regulatory channels when companies ignore their obligations. The process takes time, but enforcement is improving across the EU in 2026.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
