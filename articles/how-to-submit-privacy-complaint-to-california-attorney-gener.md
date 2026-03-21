---
layout: default
title: "Submit a Privacy Complaint to California Attorney General Under CCPA Enforcement"
description: "A practical guide for developers and power users on filing privacy complaints to the California Attorney General under CCPA enforcement. Learn the process"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-submit-privacy-complaint-to-california-attorney-general/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, privacy]
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

## Timeline and Enforcement Outcomes

Historical data shows CCPA complaints have specific patterns:

- **2020-2022**: CAG focused on major tech companies (Facebook, YouTube, Google)
- **2023-2024**: Shift toward retail and financial services
- **2025-2026**: Increased focus on AI/ML companies and data brokers

Notable settlements include:
- Meta: $100 million penalty (2022)
- Amazon: Ongoing investigation (2024+)
- Google: Multiple state coordination (2023+)

If CAG decides to investigate your company, expect:
1. Information request (30-90 days) asking for business records
2. Settlement negotiations (2-6 months)
3. Public settlement agreement with penalty and injunctive relief

The average time from complaint to settlement is 12-18 months.

## Practical Compliance Checklist for Organizations

If you're an organization handling California resident data, implement these safeguards to avoid complaints:

```python
class CCPAComplianceChecker:
    """Audit your organization's CCPA compliance"""

    def __init__(self):
        self.compliance_checks = {
            'privacy_policy': False,
            'consumer_rights_form': False,
            'request_processing': False,
            'deletion_capability': False,
            'opt_out_mechanism': False,
            'data_retention_limits': False
        }

    def verify_privacy_policy(self):
        """Privacy policy must disclose:
        - Categories of personal information collected
        - How data is used
        - Consumer rights under CCPA
        - How to submit requests
        """
        return self.privacy_policy_contains_disclosures()

    def verify_consumer_request_mechanism(self):
        """Must provide way for consumers to request:
        - Access (know)
        - Deletion
        - Opt-out of sale/sharing
        - Correction (under CPRA)
        """
        return self.has_working_request_form()

    def verify_45_day_response(self):
        """Implement automated tracking:"""
        return self.track_request_deadlines()

    def verify_no_discrimination(self):
        """Cannot penalize consumers for exercising rights"""
        return self.logs_show_no_service_changes()

    def run_audit(self):
        results = {}
        for check, method in self.compliance_checks.items():
            results[check] = getattr(self, f'verify_{check}')()
        return results

# Usage in compliance workflow
auditor = CCPAComplianceChecker()
audit_results = auditor.run_audit()
if not all(audit_results.values()):
    print("CCPA compliance gaps detected. Implement fixes immediately.")
```

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

Implement a dedicated request processor that handles all three consumer rights:

```python
from datetime import datetime, timedelta
import hashlib
import json

class CCPARequestProcessor:
    def __init__(self, database):
        self.db = database
        self.request_deadline = 45  # days

    def process_request(self, request_type, consumer_id, email):
        """Handle access, deletion, or opt-out requests"""

        # Create trackable record
        request_record = {
            'request_id': self.generate_request_id(consumer_id),
            'type': request_type,  # 'access', 'delete', 'opt_out'
            'consumer_id': consumer_id,
            'submitted_date': datetime.utcnow(),
            'deadline': datetime.utcnow() + timedelta(days=self.request_deadline),
            'status': 'received'
        }

        self.db.save_request(request_record)

        # Process based on type
        if request_type == 'access':
            return self.handle_access_request(consumer_id)
        elif request_type == 'delete':
            return self.handle_deletion_request(consumer_id)
        elif request_type == 'opt_out':
            return self.handle_opt_out(consumer_id)

    def generate_request_id(self, consumer_id):
        """Create unique, verifiable request ID"""
        timestamp = datetime.utcnow().isoformat()
        data = f"{consumer_id}{timestamp}"
        return hashlib.sha256(data.encode()).hexdigest()[:16]

    def handle_access_request(self, consumer_id):
        """Compile all personal data for consumer"""
        data = self.db.get_all_consumer_data(consumer_id)
        return json.dumps(data, indent=2)

    def handle_deletion_request(self, consumer_id):
        """Delete consumer data across all systems"""
        self.db.mark_for_deletion(consumer_id)
        self.notify_third_parties(consumer_id)  # Service providers must delete too
        return {'status': 'deletion_initiated'}

    def handle_opt_out(self, consumer_id):
        """Stop selling/sharing consumer data"""
        self.db.set_opt_out_flag(consumer_id, True)
        return {'status': 'opted_out'}
```

Ensure your systems respond to consumer requests within the mandated 45-day window. Implement automated tracking for request deadlines to avoid unintentional violations. Set up calendar alerts at day 30 and day 40 to ensure you don't miss deadlines.

{% endraw %}


## Related Reading

- [How To Use State Attorney General Office To Enforce Privacy](/privacy-tools-guide/how-to-use-state-attorney-general-office-to-enforce-privacy-/)
- [Ccpa Compliance Requirements For Online Businesses California Privacy Law](/privacy-tools-guide/ccpa-compliance-requirements-for-online-businesses-california-privacy-law-guide-2026/)
- [How To File Ftc Complaint For Privacy Violation By Company D](/privacy-tools-guide/how-to-file-ftc-complaint-for-privacy-violation-by-company-d/)
- [Enterprise Privacy Compliance Tool Comparison for GDPR.](/privacy-tools-guide/enterprise-privacy-compliance-tool-comparison-for-gdpr-and-ccpa/)
- [Cloud Storage Subpoena Risk: Provider Law Enforcement.](/privacy-tools-guide/cloud-storage-subpoena-risk-provider-law-enforcement-complia/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
