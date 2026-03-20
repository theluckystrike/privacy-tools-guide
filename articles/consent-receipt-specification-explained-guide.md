---

layout: default
title: "Consent Receipt Specification Explained: A Developer Guide"
description: "A practical guide to implementing the Consent Receipt Specification for developers and power users. Learn the JSON structure, key components, and code."
date: 2026-03-15
author: theluckystrike
permalink: /consent-receipt-specification-explained-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

The Consent Receipt Specification is a JSON-based standard from the Kantara Initiative that records exactly what a user consented to, when, under which jurisdiction, and for which purposes -- giving you a machine-readable, cryptographically verifiable proof of consent for GDPR, CCPA, and similar compliance. To implement it, you generate a signed JSON document containing the subject identifier, issuer details, granular per-service consent records, and a cryptographic signature, then store it immutably as legal evidence. This guide walks through the full JSON structure, required fields, and working Python code for generating and verifying consent receipts.

## What the Consent Receipt Specification Provides

The specification defines a JSON-based format that captures consent events with enough detail to satisfy legal requirements while remaining processable by software. Each consent receipt represents a single consent action—a user agreeing to a specific purpose or data processing activity.

Unlike vague audit logs, a consent receipt contains explicit information about what the user consented to, when they gave consent, how they provided it, and what the organization must do in response. This clarity matters for compliance audits, user rights requests, and demonstrating due diligence.

The specification builds on the ISO/IEC 29100 framework and integrates with the OpenID Connect ecosystem. It addresses a common problem: organizations collect consent but lack a reliable way to prove what users actually agreed to.

## Core Components of a Consent Receipt

Every consent receipt includes several required fields that identify the consent event, the user, and the organization requesting consent. Understanding these fields helps you design your data model correctly.

### Required Fields

The specification mandates these core elements:

- **version**: The version of the Consent Receipt Specification being used (typically "1.1" or "2.0")
- **jurisdiction**: The legal jurisdiction governing the consent
- **consent timestamp**: When the user provided consent, in ISO 8601 format
- **collection method**: How the consent was obtained (explicit, implicit, or similar)
- **subject**: Information about the user providing consent
- **issuer**: The organization receiving the consent
- **policy url**: Link to the privacy policy or consent notice
- **services**: The specific purposes or services the user consented to

### Service-Specific Consent

Each service or purpose requires its own consent record within the receipt. This structure allows granular consent management:

```json
{
  "services": [
    {
      "service": "marketing_emails",
      "purposes": ["personalized_offers", "product_updates"],
      "consent": true,
      "timestamp": "2026-03-15T10:30:00Z"
    },
    {
      "service": "analytics",
      "purposes": ["usage_analytics", "improvement_research"],
      "consent": false,
      "timestamp": "2026-03-15T10:30:00Z"
    }
  ]
}
```

This granular approach lets users consent to some activities while declining others—a requirement under GDPR's explicit consent framework.

## JSON Structure Example

Here's a complete consent receipt demonstrating the specification in practice:

```json
{
  "version": "1.1",
  "jurisdiction": "GDPR",
  "consent_timestamp": "2026-03-15T14:22:31Z",
  "collection_method": "web_form_checkbox",
  "subject": {
    "id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
    "name": "john.doe@example.com",
    "method": "email"
  },
  "issuer": {
    "name": "Example Corp",
    "contact": "privacy@example.com",
    "policy_url": "https://example.com/privacy-policy",
    "legal_url": "https://example.com/legal-notice"
  },
  "services": [
    {
      "service": "account_creation",
      "purposes": ["provide_service", "maintain_account"],
      "consent": true,
      "policy_url": "https://example.com/terms"
    }
  ],
  "salt": "aGk1ZXN0LXNhbHQtZm9yLWhhc2hpbmc=",
  "signature": {
    "algorithm": "RS256",
    "value": "eyJhbGciOiJSUzI1NiJ9..."
  }
}
```

The salt and signature fields provide integrity verification. The salt ensures the subject identifier cannot be correlated across different receipts, while the signature allows verification that the issuer actually created the receipt.

## Implementation Considerations for Developers

When implementing consent receipt generation, several practical concerns arise that the specification leaves to implementers.

### Storing Consent Receipts

Store receipts securely but accessibly. Since receipts serve as legal evidence, consider:

- Immutable storage (append-only logs or blockchain-based solutions)
- Retention policies matching applicable regulation requirements (GDPR suggests indefinite storage for consent)
- Encryption at rest with proper key management
- Regular integrity verification of stored receipts

### Generating Unique Identifiers

The subject identifier requires careful handling. Using a hash of user information with a per-organization salt prevents correlation across services while providing consistent identification:

```python
import hashlib
import secrets

def generate_subject_id(email, organization_salt):
    """Generate a privacy-preserving subject identifier."""
    normalized_email = email.lower().strip()
    preimage = f"{normalized_email}:{organization_salt}"
    hash_value = hashlib.sha256(preimage.encode()).hexdigest()
    return f"urn:uuid:{hash_value[:36]}"

# Example usage
org_salt = secrets.token_hex(16)
subject_id = generate_subject_id("user@example.com", org_salt)
print(subject_id)  # urn:uuid:550e8400-e29b-41d4-a716-446655440000
```

This approach ensures the same user gets consistent identification within your organization without exposing their email in plain text.

### Verifying Consent Receipts

Receipt verification requires checking the signature and validating the structure:

```python
import json
import jwt

def verify_consent_receipt(receipt_json, issuer_public_key):
    """Verify a consent receipt's signature and structure."""
    try:
        receipt = json.loads(receipt_json)
        
        # Verify required fields exist
        required = ['version', 'consent_timestamp', 'issuer', 'services']
        for field in required:
            if field not in receipt:
                raise ValueError(f"Missing required field: {field}")
        
        # Verify signature
        payload = jwt.decode(
            receipt['signature']['value'],
            issuer_public_key,
            algorithms=[receipt['signature']['algorithm']]
        )
        
        return True, "Valid consent receipt"
        
    except (json.JSONDecodeError, KeyError, jwt.InvalidSignatureError) as e:
        return False, f"Verification failed: {str(e)}"
```

### Integrating with User Consent Management

Modern consent management platforms (CMPs) often support the Consent Receipt Specification natively or provide adapters. If building custom infrastructure, consider:

- Webhooks for consent changes that trigger receipt generation
- API endpoints for users to retrieve their consent history
- Export functionality for data portability requirements
- Mechanisms for consent withdrawal that create new receipts marking withdrawal

## Practical Applications

The consent receipt becomes valuable in several real-world scenarios:

**Data subject access requests (DSAR)**: When users request their data, the consent receipt demonstrates exactly what they agreed to, when, and under what policy. This simplifies responding to access requests.

**Compliance audits**: Regulators increasingly request evidence of valid consent. A properly structured consent receipt provides that evidence in a standardized format.

**Cross-border data transfers**: Different jurisdictions have different consent requirements. The jurisdiction field in the receipt clarifies which legal framework applies.

**Service deprecation**: When you remove a service, existing consent receipts document which users had consented to that service and when their consent was obtained.

## Limitations to Understand

The specification records consent but does not solve every consent management challenge. It does not define how to present consent requests, how to handle consent withdrawal, or how to manage consent across complex organizational structures. These remain implementation decisions.

Additionally, the specification requires careful key management for signatures. If your signing keys are compromised, attackers could create valid-looking consent receipts. Key rotation and proper key management are essential.

## Related Reading

- [Bitwarden Vault Export Backup Guide: Complete Technical.](/privacy-tools-guide/bitwarden-vault-export-backup-guide/)
- [GDPR Joint Controller Agreement Template: A Developer Guide](/privacy-tools-guide/gdpr-joint-controller-agreement-template/)
- [VPN Warrant Canary: What It Means and Why It Matters](/privacy-tools-guide/vpn-warrant-canary-what-it-means/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
