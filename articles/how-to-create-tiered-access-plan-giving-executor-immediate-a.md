---


layout: default
title: "How to Create Tiered Access Plan Giving Executor Immediate Access and Heirs Delayed Access"
description: "Learn how to build a tiered digital estate system where executors get immediate access to critical accounts while heirs receive delayed access. Includes code examples and practical implementation guides for developers."
date: 2026-03-16
author: "Privacy Tools Guide"
permalink: /how-to-create-tiered-access-plan-giving-executor-immediate-a/
reviewed: true
score: 8
categories: [guides]
---


{% raw %}

Digital estate planning requires careful consideration of access timing. Many people want their executor to manage affairs immediately after death while restricting heir access until after probate or a waiting period. This guide shows developers how to build a tiered access system that enforces these temporal constraints programmatically.

## Understanding the Tiered Access Model

A tiered access plan separates digital assets into layers based on when they should become accessible. The executor layer grants immediate access to accounts needed for funeral arrangements, debt payment, and estate administration. The heir layer delays access to sentimental items, inheritance accounts, and less critical digital property.

This approach solves several real problems. Executors need quick access to banking and utility accounts to prevent service interruptions. Heirs may need time to grieve before handling financial matters. Some jurisdictions require probate court approval before certain assets transfer, making immediate access inappropriate.

### Core Components of the System

The implementation requires three main components:

1. **Access Controller** — A service that verifies identity and enforces time-based rules
2. **Encrypted Vault** — Secure storage for credentials with conditional decryption
3. **Notification System** — Mechanisms to alert parties when access conditions are met

## Implementation Architecture

### The Access Controller

The access controller maintains a JSON configuration that defines access tiers and their activation conditions:

```javascript
// config/access-tiers.json
{
  "estate_id": "unique-estate-identifier",
  "tiers": [
    {
      "tier_id": "executor_immediate",
      "name": "Executor Full Access",
      "role": "executor",
      "activation": {
        "type": "immediate",
        "conditions": ["death_verified", "appointment_confirmed"]
      },
      "resources": [
        "banking_primary",
        "utilities_management",
        "insurance_claims",
        "subscription_cancellation"
      ]
    },
    {
      "tier_id": "heirs_delayed",
      "name": "Heir Inheritance Access",
      "role": "heir",
      "activation": {
        "type": "delayed",
        "delay_days": 30,
        "conditions": ["probate_completed", "executor_approval"]
      },
      "resources": [
        "investment_accounts",
        "digital_photos",
        "personal_documents",
        "social_media_estate"
      ]
    }
  ]
}
```

This configuration allows developers to customize access windows based on specific requirements.

### Time-Locked Encryption

For sensitive credentials that must remain inaccessible until conditions are met, implement time-locked encryption using a key derivation function with a built-in delay:

```python
import hashlib
import time
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
from cryptography.hazmat.backends import default_backend

class TimeLockedVault:
    def __init__(self, salt, delay_seconds=2592000):  # 30 days default
        self.salt = salt
        self.delay_seconds = delay_seconds
        self.locked_at = None
    
    def lock(self):
        """Initialize the time lock"""
        self.locked_at = time.time()
    
    def unlock(self, current_time=None):
        """Attempt to unlock the vault"""
        if current_time is None:
            current_time = time.time()
        
        elapsed = current_time - self.locked_at
        if elapsed >= self.delay_seconds:
            return True
        else:
            remaining = self.delay_seconds - elapsed
            print(f"Vault locked. {remaining/86400:.1f} days remaining")
            return False
    
    def derive_key(self, password):
        """Derive encryption key from password"""
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=self.salt,
            iterations=100000,
            backend=default_backend()
        )
        return kdf.derive(password.encode())
```

This approach adds computational delays that make brute-force attacks impractical while allowing legitimate access after the waiting period.

### The Verification Service

When someone requests access, the verification service checks multiple conditions before granting it:

```javascript
// services/access-verifier.js
class AccessVerifier {
  constructor(config) {
    this.config = config;
  }
  
  async verifyAccess(request) {
    const { tier_id, claimant_id, evidence } = request;
    const tier = this.config.tiers.find(t => t.tier_id === tier_id);
    
    if (!tier) {
      throw new Error('Invalid tier requested');
    }
    
    // Check activation conditions
    for (const condition of tier.activation.conditions) {
      const satisfied = await this.checkCondition(condition, evidence);
      if (!satisfied) {
        return { granted: false, reason: `Condition not met: ${condition}` };
      }
    }
    
    // Handle delayed access tiers
    if (tier.activation.type === 'delayed') {
      const unlockTime = await this.calculateUnlockTime(tier);
      if (Date.now() < unlockTime) {
        return {
          granted: false,
          reason: 'Access pending',
          unlock_date: new Date(unlockTime).toISOString()
        };
      }
    }
    
    return { granted: true, resources: tier.resources };
  }
  
  async checkCondition(condition, evidence) {
    switch (condition) {
      case 'death_verified':
        return this.verifyDeathCertificate(evidence.certificate);
      case 'appointment_confirmed':
        return this.verifyExecutorAppointment(evidence.probate_documents);
      case 'probate_completed':
        return this.checkProbateStatus(evidence.court_records);
      case 'executor_approval':
        return this.verifyExecutorSignature(evidence.approval_token);
      default:
        return false;
    }
  }
}
```

## Practical Deployment Strategies

### Using Password Manager Enterprise Features

Several enterprise password managers now support inheritance features that integrate with this model. Bitwarden's Collections and Access Policies can enforce tiered access:

```yaml
# Bitwarden organization policy configuration
policies:
  - name: "Executor Collection Access"
    enabled: true
    collections:
      - "Estate Administration"
    requirements:
      - type: "user_group"
        group: "appointed_executor"
      - type: "mfa_required"
  
  - name: "Heir Delayed Collection"
    enabled: true
    collections:
      - "Inheritance Assets"
    requirements:
      - type: "temporal_gate"
        delay_days: 30
        trigger: "executor_release"
```

### Self-Hosted Implementation with Vaultwarden

For developers preferring self-hosted solutions, Vaultwarden (formerly Bitwarden RS) can be configured with custom plugins for temporal access control:

```bash
# Docker compose for Vaultwarden with temporal access extension
version: '3.8'
services:
  vaultwarden:
    image: vaultwarden/server:latest
    volumes:
      - ./vw-data:/data
      - ./plugins/temporal-access:/plugins
    environment:
      - EXTENSIONS_ENABLED=temporal_access
      - TEMPORAL_GRACE_PERIOD_DAYS=30
    ports:
      - "8080:80"
```

### Hardware Security Key Integration

For higher security, integrate hardware security keys (YubiKey, SoloKey) into the access verification process. The executor receives a key that must be present for immediate-tier access, while heir keys are programmed to activate only after the delay period.

```python
from fido2.server import Fido2Server
from fido2.webauthn import PublicKeyCredentialCreationOptions

class HardwareKeyAccess:
    def __init__(self, rp_id):
        self.server = Fido2Server(rp_id)
    
    def register_executor_key(self, credential_data):
        """Register executor hardware key with elevated permissions"""
        # Executor keys get immediate access flag
        return self.server.register(credential_data, flags=['immediate_access'])
    
    def register_heir_key(self, credential_data, unlock_date):
        """Register heir key with time-based activation"""
        # Heir keys store unlock date in credential metadata
        return self.server.register(
            credential_data,
            metadata={'unlock_date': unlock_date, 'tier': 'heir'}
        )
    
    def authenticate(self, credential, assertion):
        """Verify hardware key and check temporal constraints"""
        result = self.server.authenticate(credential, assertion)
        
        if result.flags.get('immediate_access'):
            return {'access': 'granted', 'tier': 'executor'}
        
        unlock_date = result.metadata.get('unlock_date')
        if unlock_date and datetime.now() >= unlock_date:
            return {'access': 'granted', 'tier': 'heir'}
        
        return {'access': 'denied', 'reason': 'pending_unlock_date'}
```

## Best Practices for Configuration

When implementing a tiered access system, consider these developer recommendations:

**Document everything** — Create a separate configuration guide that explains each tier to executors and heirs. The technical implementation should have human-readable documentation.

**Test the system while alive** — Use a "test mode" that allows you to verify all access paths work correctly without triggering actual estate conditions.

**Plan for key recovery** — Ensure there's a mechanism for key replacement if hardware tokens fail or passwords are forgotten. This typically involves a trusted contact or attorney.

**Consider jurisdiction requirements** — Some countries have specific probate timelines. Adjust delay periods to accommodate local laws.

**Maintain audit logs** — Every access attempt should be logged with timestamps, successful or not. This provides accountability and helps resolve disputes.

A well-designed tiered access system provides the flexibility to manage digital estates according to family needs and legal requirements. The key is balancing security with practicality—ensuring executors can act quickly while respecting the intended timeline for heir access.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
