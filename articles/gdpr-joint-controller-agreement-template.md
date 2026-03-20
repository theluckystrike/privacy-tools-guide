---
layout: default
title: "GDPR Joint Controller Agreement Template: A Practical."
description: "Learn how to create a compliant GDPR joint controller agreement with this practical template and code examples designed for developers and technical teams."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /gdpr-joint-controller-agreement-template/
intent-checked: true
voice-checked: true
reviewed: true
score: 8
categories: [guides]
---

{% raw %}
When multiple organizations process personal data together, GDPR requires clarity on who does what. A joint controller agreement formalizes these responsibilities, preventing the blame game when things go wrong. This guide provides a practical template you can adapt for your projects.

## What Is a Joint Controller Agreement?

Under GDPR (Article 26), two or more controllers can determine the purposes and means of processing jointly. When this happens, they must establish "joint controllership" through a legal agreement. The agreement doesn't reduce liability—each party remains responsible for compliance—but it defines internal responsibilities.

For developers, this often arises in scenarios like:
- Analytics platforms sharing data with third-party tools
- Multi-tenant SaaS applications processing data for different customers
- Mobile apps integrating SDKs that collect user data
- API integrations between services

## Key Elements Required Under GDPR

Your joint controller agreement must include:

1. **Purpose of processing** — What data you're collecting and why
2. **Categories of data** — Personal data types involved
3. **Data subjects** — Whose data (customers, employees, etc.)
4. **Responsibilities matrix** — Who handles what
5. **Data protection measures** — Technical and organizational controls
6. **Sub-processor management** — How you'll handle vendors
7. **Breach notification procedures** — Who notifies whom and when
8. **Contact points** — How data subjects can reach you

## Practical Template

```markdown
# Joint Controller Agreement

**Effective Date:** [DATE]
**Parties:** [Organization A] and [Organization B]

## 1. Purpose and Scope
This agreement governs the joint processing of personal data in connection with [PRODUCT/SERVICE NAME].

### 1.1 Purpose
The parties jointly determine the purposes and means of processing personal data for [specific use case].

### 1.2 Categories of Data
- Name, email, IP address
- [Other data types]

## 2. Responsibilities

| Task | Organization A | Organization B |
|------|----------------|----------------|
| Data Collection | Primary | Secondary |
| Data Storage | Primary | None |
| Security Measures | Primary | Secondary |
| Data Subject Requests | Shared | Shared |
| Breach Notification | Primary | Secondary |

### 2.1 Organization A Responsibilities
- Implement technical security measures
- Maintain processing records
- Handle data subject access requests
- Notify Organization B within 24 hours of any breach

### 2.2 Organization B Responsibilities
- Ensure lawful basis for data collection
- Support data subject requests within 5 business days
- Maintain adequate security for its systems

## 3. Data Protection Measures
Both parties implement:
- Encryption in transit (TLS 1.3)
- Encryption at rest (AES-256)
- Access controls and logging
- Regular security audits

## 4. Data Subject Rights
Both parties commit to responding to data subject requests within 30 days. For requests received by one party, that party will forward to the other within 48 hours.

## 5. Breach Notification
- Initial notification: within 24 hours of discovery
- Full report: within 72 hours
- Contact: [email@company.com]

## 6. Term and Termination
This agreement remains in effect for the duration of the data processing relationship. Upon termination, data must be returned or deleted within 30 days.

## 7. Governing Law
This agreement is governed by [jurisdiction] law.

**Signed:**
_________________     _________________
[Organization A]      [Organization B]
```

## Code Example: Mapping Responsibilities

For developers building tools that generate or manage these agreements, here's a simple data structure:

```javascript
const jointControllerAgreement = {
  parties: [
    {
      name: "Acme Corp",
      role: "primary",
      responsibilities: [
        "data_collection",
        "secure_storage",
        "breach_notification"
      ],
      contact: "privacy@acme.com"
    },
    {
      name: "Partner Inc",
      role: "secondary", 
      responsibilities: [
        "data_analysis",
        "support_requests"
      ],
      contact: "dpo@partner.com"
    }
  ],
  processing: {
    purpose: "analytics_and_improvement",
    categories: [guides],
    legal_basis: "legitimate_interest"
  },
  timeline: {
    breach_notification_hours: 24,
    dsar_response_days: 30,
    data_deletion_days: 30
  }
};
```

## Implementation Checklist

Before finalizing your agreement, verify:

- [ ] Each party has designated a privacy contact
- [ ] Data flows are documented (consider a data flow diagram)
- [ ] Security measures meet GDPR Article 32 requirements
- [ ] Sub-processor agreements are in place if using vendors
- [ ] Cross-border transfer mechanisms are specified (if applicable)
- [ ] Review cycle is established (annually recommended)

## Common Pitfalls to Avoid

Vague responsibility definitions are a common problem. Avoid generic statements like "both parties ensure compliance" and be specific about who does what.

Missing breach procedures are another gap to watch for. GDPR requires notification within 72 hours of becoming aware of a breach, so specify exact timelines and communication channels in the agreement.

Third-party SDKs are easy to overlook. If you embed analytics or advertising SDKs, you may need to document whether you're a joint controller with that vendor. Many SDKs have their own terms—review them carefully.

## Automating Agreement Management

For larger organizations, consider storing these agreements in a structured format:

```json
{
  "agreement_id": "JCA-2026-001",
  "status": "active",
  "effective_date": "2026-01-01",
  "review_date": "2027-01-01",
  "parties": [...],
  "data_processing": {...},
  "last_updated": "2026-03-15"
}
```

This enables you to query active agreements, track expiration dates, and generate compliance reports programmatically.

## When to Seek Legal Counsel

This template provides a starting point, but complex processing activities, cross-border data transfers, or high-risk processing may require professional legal advice. The ICO (in the UK) and other supervisory authorities provide guidance on joint controllership that complements this template.

---

A well-drafted joint controller agreement protects all parties and, more importantly, protects the data subjects whose information you're handling. Take time to get it right—it'll save headaches later.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
