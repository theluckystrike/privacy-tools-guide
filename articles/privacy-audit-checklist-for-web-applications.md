---
layout: default
title: "Privacy Audit Checklist for Web Applications: A Developer"
description: "A practical privacy audit checklist for web applications with implementation code. Covers data collection audit, consent management, encryption requirements, and compliance verification."
date: 2026-03-15
author: theluckystrike
permalink: /privacy-audit-checklist-for-web-applications/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
---

{% raw %}
# Privacy Audit Checklist for Web Applications: A Developer

Conducting a privacy audit for web applications requires systematically evaluating data collection practices, consent mechanisms, storage security, and regulatory compliance. This checklist provides actionable items with implementation examples to help developers secure their applications.

## 1. Data Collection Audit

Before implementing any privacy controls, document what data you collect and why.

### Inventory Data Flows

```python
# Example: Data inventory tracking
class DataAsset:
    def __init__(self, name, category, pii, retention_period, legal_basis):
        self.name = name
        self.category = category  # user_input, analytics, technical
        self.pii = pii  # boolean - personally identifiable?
        self.retention_period = retention_period
        self.legal_basis = legal_basis  # consent, contract, legitimate_interest

# Document your data assets
data_inventory = [
    DataAsset("email", "user_input", True, "until_withdrawal", "consent"),
    DataAsset("ip_address", "technical", True, "90_days", "legitimate_interest"),
    DataAsset("session_token", "technical", False, "session", "contract"),
]
```

Check each endpoint for:
- Unnecessary data collection
- PII in logs or analytics
- Third-party data sharing
- Data retention periods

## 2. Consent Management Implementation

### Consent Collection API

```javascript
// Frontend: Consent collection with proof of consent
async function collectConsent(consentTypes, purpose) {
  const consentRecord = {
    user_id: getAnonymousId(),
    timestamp: new Date().toISOString(),
    ip_address: await getClientIP(),
    user_agent: navigator.userAgent,
    consent_types: consentTypes,
    purpose: purpose,
    version: '1.0',
    obtained_via: 'explicit_checkbox'
  };
  
  // Hash for integrity verification
  const consent_hash = await sha256(JSON.stringify(consentRecord));
  consentRecord.hash = consent_hash;
  
  // Store locally as proof
  localStorage.setItem('consent_proof', JSON.stringify(consentRecord));
  
  // Send to backend
  await fetch('/api/consent', {
    method: 'POST',
    body: JSON.stringify(consentRecord)
  });
  
  return consentRecord;
}

// Usage
document.getElementById('signup-form').addEventListener('submit', async (e) => {
  const marketing_consent = document.getElementById('marketing-consent').checked;
  const analytics_consent = document.getElementById('analytics-consent').checked;
  
  await collectConsent(
    ['marketing', 'analytics'].filter(c => 
      c === 'marketing' && marketing_consent || 
      c === 'analytics' && analytics_consent
    ),
    'account_creation'
  );
});
```

### Backend Consent Storage

```python
from dataclasses import dataclass
from datetime import datetime
from typing import List
import hashlib
import json

@dataclass
class ConsentRecord:
    user_id: str
    consent_types: List[str]
    timestamp: datetime
    ip_address: str
    user_agent: str
    purpose: str
    version: str
    hash: str

class ConsentStore:
    def __init__(self):
        self.consents = {}  # In production, use a database
    
    def record_consent(self, record: ConsentRecord) -> bool:
        # Verify consent integrity
        expected_hash = self._compute_hash(record)
        if expected_hash != record.hash:
            return False
        
        # Store consent with version tracking
        if record.user_id not in self.consents:
            self.consents[record.user_id] = []
        
        self.consents[record.user_id].append({
            'consent_types': record.consent_types,
            'timestamp': record.timestamp.isoformat(),
            'purpose': record.purpose,
            'version': record.version
        })
        return True
    
    def has_valid_consent(self, user_id: str, consent_type: str) -> bool:
        if user_id not in self.consents:
            return False
        
        # Check most recent consent for type
        consents = self.consents[user_id]
        if not consents:
            return False
        
        latest = consents[-1]
        return consent_type in latest.get('consent_types', [])
```

## 3. Encryption Requirements

### Data Encryption at Rest

```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2
import base64
import os

class EncryptionManager:
    def __init__(self, master_key: str):
        self.key = self._derive_key(master_key)
        self.cipher = Fernet(self.key)
    
    def _derive_key(self, master_key: str) -> bytes:
        salt = os.urandom(16)  # In production, store salt securely
        kdf = PBKDF2(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100000,
        )
        key = base64.urlsafe_b64encode(kdf.derive(master_key.encode()))
        return key
    
    def encrypt(self, data: str) -> str:
        return self.cipher.encrypt(data.encode()).decode()
    
    def decrypt(self, encrypted_data: str) -> str:
        return self.cipher.decrypt(encrypted_data.encode()).decode()

# Usage
encryption = EncryptionManager(os.environ['MASTER_KEY'])

# Encrypt PII before storage
user_email_encrypted = encryption.encrypt("user@example.com")
# Store encrypted_email in database
```

### TLS Configuration Verification

```nginx
# Nginx SSL configuration for production
server {
    listen 443 ssl http2;
    server_name example.com;
    
    # Modern TLS configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    
    # HSTS header
    add_header Strict-Transport-Security "max-age=63072000" always;
    
    # Additional security headers
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
}
```

## 4. Data Subject Rights Implementation

### Right to Erasure (Right to be Forgotten)

```python
from datetime import datetime
from typing import Dict, List
import asyncio

class DataSubjectRights:
    def __init__(self, db_connection):
        self.db = db_connection
    
    async def handle_erasure_request(self, user_id: str) -> Dict:
        # Verify identity first
        if not await self._verify_identity(user_id):
            return {'status': 'identity_verification_required'}
        
        # Collect all data locations
        data_locations = await self._find_user_data(user_id)
        
        # Execute erasure from each location
        erasure_results = []
        for location in data_locations:
            result = await self._erase_from_location(user_id, location)
            erasure_results.append(result)
        
        # Create audit trail
        await self._create_erasure_audit(user_id, erasure_results)
        
        return {
            'status': 'completed',
            'erased_from': data_locations,
            'timestamp': datetime.utcnow().isoformat()
        }
    
    async def _find_user_data(self, user_id: str) -> List[Dict]:
        locations = []
        
        # Check main user table
        locations.append({'table': 'users', 'id': user_id})
        
        # Check analytics data
        locations.append({'table': 'analytics_events', 'user_id': user_id})
        
        # Check logs (mark for deletion)
        locations.append({'table': 'application_logs', 'user_id': user_id})
        
        return locations
    
    async def _erase_from_location(self, user_id: str, location: Dict) -> bool:
        table = location['table']
        # Actual deletion or anonymization
        query = f"DELETE FROM {table} WHERE user_id = ?"
        await self.db.execute(query, (user_id,))
        return True
```

### Data Export (Portability)

```python
import json
from datetime import datetime

class DataPortability:
    def export_user_data(self, user_id: str) -> Dict:
        export_data = {
            'export_timestamp': datetime.utcnow().isoformat(),
            'user_id': user_id,
            'data': {}
        }
        
        # Collect all user data
        export_data['data']['profile'] = self.get_profile(user_id)
        export_data['data']['activity'] = self.get_activity(user_id)
        export_data['data']['preferences'] = self.get_preferences(user_id)
        export_data['data']['consents'] = self.get_consent_history(user_id)
        
        return export_data
    
    def format_as_json(self, user_id: str) -> str:
        data = self.export_user_data(user_id)
        return json.dumps(data, indent=2)
```

## 5. Security Checklist

- [ ] Implement rate limiting on all endpoints
- [ ] Add CSRF protection tokens
- [ ] Configure Content Security Policy headers
- [ ] Set secure cookie flags (HttpOnly, Secure, SameSite)
- [ ] Implement input validation and sanitization
- [ ] Enable audit logging for data access
- [ ] Configure automated vulnerability scanning
- [ ] Review third-party library dependencies
- [ ] Implement session timeout mechanisms
- [ ] Enable account takeover detection

## 6. Compliance Verification

```bash
# Automated compliance checks using Python
#!/usr/bin/env python3
import subprocess
import json

def run_compliance_checks():
    checks = []
    
    # Check SSL certificate validity
    result = subprocess.run(
        ['openssl', 's_client', '-connect', 'example.com:443', '-showcerts'],
        capture_output=True
    )
    checks.append({
        'check': 'ssl_certificate',
        'passed': result.returncode == 0,
        'details': 'Certificate valid' if result.returncode == 0 else 'Certificate issue'
    })
    
    # Check security headers
    result = subprocess.run(
        ['curl', '-I', 'https://example.com'],
        capture_output=True
    )
    headers = result.stdout.decode()
    checks.append({
        'check': 'hsts_header',
        'passed': 'Strict-Transport-Security' in headers
    })
    
    return checks
```

Run these checks periodically and maintain evidence of compliance for regulatory audits.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
