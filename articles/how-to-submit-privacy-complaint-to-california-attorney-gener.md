---
layout: default
title: "How to Submit a Privacy Complaint to California Attorney General Under CCPA Enforcement"
description: "A practical guide for developers and power users on filing privacy complaints to the California Attorney General under CCPA enforcement. Learn the process, requirements, and tools for asserting your consumer rights."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-submit-privacy-complaint-to-california-attorney-general/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

The California Consumer Privacy Act (CCPA) grants California residents significant rights over their personal information. When businesses violate these rights, the California Attorney General (CAG) can enforce penalties up to $2,500 per unintentional violation and $7,500 per intentional violation. This guide walks you through submitting a privacy complaint to the CAG, with practical examples tailored for developers and power users who understand the technical aspects of data privacy.

## Understanding CCPA Enforcement Authority

The CAG holds primary enforcement authority for CCPA violations. Unlike some privacy laws that rely solely on private lawsuits, the CCPA enables the Attorney General to take action against businesses that fail to comply with consumer rights requirements. This makes the CAG complaint process a powerful tool for accountability.

Before filing, ensure your complaint involves a for-profit business that collects California residents' personal information and meets one of these thresholds: annual gross revenue exceeding $25 million, buys or sells personal information of 100,000+ consumers, or derives 50% or more of annual revenue from selling personal information.

## Prerequisites for Filing a Complaint

Gather documentation before submitting your complaint. The CAG needs specific evidence to evaluate your case effectively.

**Consumer Rights Violations to Report**:
- Business failed to respond to a request to know within 45 days
- Business refused to delete personal information without valid justification
- Business sold your data without providing opt-out mechanisms
- Business discriminatory pricing or service differences for exercising CCPA rights
- Business collected information not disclosed in their privacy policy

Document every interaction with the business. Save emails, screenshots of web forms, and timestamps of requests. If you submitted a request programmatically, log your API calls and responses.

## Submitting Your Complaint Online

The CAG provides an online complaint portal at oag.ca.gov/contact/consumer-complaint-against-business-or-company. Navigate to this page and select the appropriate category related to your complaint type.

The complaint form requires:
1. Your personal information (name, address, email, phone)
2. Business name and contact information
3. Detailed description of the violation
4. Supporting documentation upload
5. Declaration that information is accurate

For developers submitting requests programmatically, the 45-day response window is critical. If a business ignores API-based requests or returns errors without proper justification, document this in your complaint. Reference the specific API endpoints, HTTP status codes, and response bodies.

## Writing an Effective Complaint Description

Structure your complaint description for maximum impact. The CAG reviews thousands of complaints—clarity and specificity improve your chances of action.

### Effective Complaint Template

```
COMPLAINT DESCRIPTION:

1. Consumer Request Submitted:
   - Date: [YYYY-MM-DD]
   - Method: [Email/Web Form/API/Phone]
   - Request Type: [Right to Know/Delete/Opt-Out]
   - Request Reference Number: [if available]

2. Business Response:
   - Initial Response Date: [YYYY-MM-DD]
   - Response Content: [Summary of what business said]
   - Final Outcome: [How request was resolved or ignored]

3. Specific Violation:
   - CCPA Section: [e.g., Section 1798.100]
   - Violation Description: [Clear explanation]

4. Supporting Evidence:
   - [List attached documents]
```

### Code Snippet: Logging Your Request

If you submit requests programmatically, maintain structured logs:

```python
import logging
import json
from datetime import datetime
import hashlib

class CCPARequestLogger:
    def __init__(self, business_id):
        self.business_id = business_id
        self.logger = logging.getLogger(f"ccpa_{business_id}")
        self.logger.setLevel(logging.DEBUG)
        
    def log_request(self, request_type, payload, response):
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "business_id": self.business_id,
            "request_type": request_type,
            "payload_hash": hashlib.sha256(
                json.dumps(payload, sort_keys=True).encode()
            ).hexdigest()[:16],
            "response_status": response.status_code,
            "response_body_hash": hashlib.sha256(
                response.text.encode()
            ).hexdigest()[:16]
        }
        self.logger.info(json.dumps(log_entry))
        
    def generate_complaint_doc(self):
        # Export logs for complaint documentation
        pass
```

This logger captures essential data for potential complaints, including payload hashes that prove what you sent without storing sensitive request contents.

## What Happens After Submission

After submitting your complaint, expect the following process:

**Initial Review (2-4 weeks)**: The CAG acknowledges receipt and conducts a preliminary review to determine if the complaint falls within their enforcement scope.

**Investigation (Variable)**: If the CAG decides to investigate, they may contact you for additional information. Not all complaints result in investigation—prioritization depends on pattern complaints and resource availability.

**Resolution Outcomes**:
- Business receives warning letter
- Business enters settlement negotiations
- Business faces civil prosecution
- Case referred to other agencies

You will receive updates on case status via email. However, the CAG does not disclose the outcome of enforcement actions to individual complainants in detail.

## Alternative: California Privacy Rights Act (CPRA)

California residents gained additional rights under the CPRA, which amended and expanded the CCPA effective January 1, 2023. Key additions include:
- Right to correct inaccurate personal information
- Right to limit use of sensitive personal information
- California Privacy Protection Agency (CPPA) with expanded authority

For CPRA-specific violations, you can also file complaints with the California Privacy Protection Agency at privacy.ca.gov.

## Developer-Specific Considerations

If you're building applications that handle California consumer data, understand that users may file complaints against your organization. Implement CCPA compliance:

```javascript
// Example: CCPA opt-out response headers
app.use('/api/*', (req, res, next) => {
  const userDoNotSell = req.headers['x-ccpa-do-not-sell'];
  if (userDoNotSell === 'true') {
    // Process opt-out request
    res.setHeader('X-CCPA-Do-Not-Sell-Confirmed', 'true');
    res.setHeader('Content-Type', 'application/json');
    return res.status(200).json({
      status: 'opted-out',
      timestamp: new Date().toISOString()
    });
  }
  next();
});
```

Ensure your systems respond to consumer requests within the mandated 45-day window. Implement automated tracking for request deadlines to avoid unintentional violations.

{% endraw %}
## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
