---
layout: default
title: "How To File Gdpr Complaint Against Company That Refuses"
description: "A technical guide for developers and power users on exercising GDPR data deletion rights, escalating complaints, and using regulatory tools when"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-file-gdpr-complaint-against-company-that-refuses-to-d/
categories: [guides]
tags: [privacy-tools-guide, tools]
reviewed: true
score: 9
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

## Step 4: Use Technical Tools

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

## Table of Contents

- [Automation Tools and Open-Source Solutions](#automation-tools-and-open-source-solutions)
- [Submitted Requests](#submitted-requests)
- [Follow-ups Required](#follow-ups-required)
- [Legal Framework Deep Dive](#legal-framework-deep-dive)
- [Building Escalation Strategy](#building-escalation-strategy)
- [Case Study: Real Company Responses](#case-study-real-company-responses)

## Automation Tools and Open-Source Solutions

### GDPR Request Automation

Several open-source tools automate the GDPR request process:

```bash
# Install GDPR CLI tool
pip install gdpr-cli

# Generate request templates
gdpr-cli generate-request --company facebook --request-type deletion

# Track requests across multiple companies
gdpr-cli track --format json > requests_log.json
```

**GDPR CLI Capabilities:**

- Generates customized request templates for specific companies
- Maintains request tracking database
- Provides company-specific email addresses for privacy officers
- Tracks request deadlines and follow-up dates

### Database of Company Privacy Contacts

Maintain a local CSV of companies and their privacy contacts to streamline future requests:

```csv
company_name,privacy_email,dpa_authority,jurisdiction,last_requested
Facebook,privacy@facebook.com,EDPB,EU,2026-01-15
Google,privacy-policy@google.com,BfDI,Germany,2026-01-10
Amazon,privacy@amazon.com,ICO,UK,2026-02-01
```

Load into a Python script for bulk request generation:

```python
import csv
import smtplib
from datetime import datetime

def send_gdpr_requests(csv_file):
    with open(csv_file, 'r') as f:
        reader = csv.DictReader(f)
        for row in reader:
            email_body = f"""
Dear {row['company_name']} Privacy Officer,

I am requesting deletion of my personal data under GDPR Article 17.
This request was submitted on {datetime.now().strftime('%Y-%m-%d')}.
Request ID: {uuid.uuid4()}

I expect your response within 30 days as required by GDPR Article 17(3).

Regards,
[Your Name]
            """
            # Send email implementation here
```

### Document Management System

Create a structured system to organize all GDPR-related documentation:

```bash
# Create directory structure
mkdir -p gdpr-requests/{companies,responses,evidence,appeals}

# Track with version control
git init
git add .
git commit -m "GDPR request management initialized"

# Create manifest file
cat > MANIFEST.md << 'EOF'
# GDPR Request Tracker

## Submitted Requests
| Company | Date | Request Type | Reference ID | Status |
|---------|------|--------------|--------------|--------|
| Facebook | 2026-01-15 | Data Access | REQ-001 | Received |
| Google | 2026-01-20 | Deletion | REQ-002 | Pending |

## Follow-ups Required
- Facebook: No response after 15 days
- Google: Response received, needs analysis
EOF
```

## Legal Framework Deep Dive

### GDPR Article 17 Requirements

Companies cannot refuse deletion when:

1. **Data no longer necessary** - You only used service once, data serves no purpose
2. **Consent withdrawn** - You explicitly withdraw consent and no legal basis remains
3. **Unlawful processing** - Company violated GDPR rules when collecting data
4. **Legal obligation** - A regulatory requirement to delete (e.g., children's data under COPPA)
5. **Legitimate objection** - You object to processing and legitimate basis doesn't override it

**Valid refusal reasons:**

- **Legal compliance**: Court order, regulatory requirement, or taxation law requires retention
- **Establishing/defending legal claims**: Company may need data for ongoing litigation
- **Public interest**: Rarely applies outside government agencies
- **Archive or research**: If anonymized

### International Jurisdiction Considerations

**EU Member States:** File with your DPA (Datenschutzbehörde, CNIL, ICO, etc.)

**UK (post-Brexit):** File with Information Commissioner's Office (ICO)

**Switzerland:** File with FDPIC (Federal Data Protection and Information Commissioner)

**California (CCPA):** File with California Attorney General or private right of action

**Canada (PIPEDA):** File with Office of the Privacy Commissioner

Most DPAs have expedited processes for "urgent" matters where companies clearly violate rights.

## Building Escalation Strategy

### Three-Tier Escalation Framework

**Tier 1: Direct Company Contact (30 days)**
- Send deletion request to privacy@company.com
- Keep timestamped copy
- Request read receipt confirmation

**Tier 2: Formal Escalation (30 days)**
- Escalate to Data Protection Officer or Legal
- Reference GDPR articles specifically
- Set 30-day response deadline
- Warn of DPA complaint intent

**Tier 3: Regulatory Complaint (submit to DPA)**
- File with appropriate supervisory authority
- Include all previous correspondence
- Provide evidence of company's refusal
- DPA investigates and enforces compliance

### Template Escalation Email

```
Subject: GDPR Article 17 Escalation - Request ID: [REFERENCE]

To the Data Protection Officer:

I submitted a GDPR Article 17 deletion request on [DATE] with reference ID [REF-001].
Your company responded on [DATE] with refusal reasoning that I believe violates GDPR.

Specific violations:
- [Point 1]: GDPR requires [specific article], your response [violates this by doing X]
- [Point 2]: [Similar analysis]

I am escalating this matter and expect your response within 30 days.
Failure to provide valid justification will result in a complaint to [DPA NAME].

Documentation:
- Original request: [attached]
- Company response: [attached]
- Analysis of violation: [attached]

Regards,
[Your Name]
[Your Email]
[Account ID if applicable]
```

## Case Study: Real Company Responses

### Common Company Refusal Patterns

**Pattern 1: Vague Legal Basis**
- Company claim: "We must retain for legal compliance"
- Reality: They haven't identified specific law
- Response: Demand they cite the specific regulation

**Pattern 2: Contractual Retention**
- Company claim: "Data retention is required by contract"
- Reality: Their internal contract with you (terms of service)
- Response: Terms of service cannot override GDPR rights

**Pattern 3: Legitimate Interest Claim**
- Company claim: "Our legitimate business interest requires retention"
- Reality: They haven't balanced your privacy rights against their interest
- Response: Demand impact assessment (DPIA) showing why their interest outweighs yours

## Frequently Asked Questions

**How long does it take to file gdpr complaint against company that refuses?**

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

- [Gdpr Right To Erasure How To Force Companies To Delete All](/gdpr-right-to-erasure-how-to-force-companies-to-delete-all-y/)
- [How To File Ftc Complaint For Privacy Violation By Company](/how-to-file-ftc-complaint-for-privacy-violation-by-company-d/)
- [How To Request Data Deletion From Companies Not Covered](/how-to-request-data-deletion-from-companies-not-covered-by-g/)
- [GDPR Article 17 Erasure Implementation](/gdpr-article-17-erasure-implementation-code/)
- [Gdpr Data Breach Notification Rights What Company Must](/gdpr-data-breach-notification-rights-what-company-must-tell-you-within-seventy-two-hours/)
- [Claude vs ChatGPT for Drafting Gdpr Compliant Privacy](https://bestremotetools.com/claude-vs-chatgpt-for-drafting-gdpr-compliant-privacy-polici/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
