---
layout: default
title: "WebAuthn Implementation Guide for Developers"
description: "A practical developer guide to implementing WebAuthn authentication. Code examples for registration, authentication, and best practices."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /webauthn-implementation-guide-developers/
reviewed: true
score: 8
categories: [guides]
---

WebAuthn (Web Authentication) is a W3C standard that enables secure passwordless authentication using public-key cryptography. This guide covers implementation details for developers building WebAuthn into their applications.

## Understanding WebAuthn Fundamentals

WebAuthn replaces passwords with cryptographic key pairs. When users register, their device generates a public-private key pair. The private key stays on their authenticator (security key, smartphone, or biometric system), while the public key gets stored on your server. During authentication, the server sends a challenge that the user signs with their private key.

The flow involves three main operations: registration, authentication, and credential management. Each operation uses specific JavaScript APIs on the client side and corresponding server endpoints.

## Server-Side Challenge Generation

Your server initiates every WebAuthn flow by generating a challenge. This challenge must be cryptographically random and stored temporarily for verification.

```python
import secrets
import base64

def generate_challenge():
    """Generate a 32-byte random challenge."""
    challenge = secrets.token_bytes(32)
    return base64.urlsafe_b64encode(challenge).rstrip(b'=')
```

Store the challenge in your session or temporary storage with an expiration. The client returns this challenge signed, and you verify the signature matches.

## Registration Implementation

The registration flow has two phases: server prepares options, then client creates the credential.

### Server: Generate Registration Options

```python
import json
import uuid
from datetime import datetime, timedelta

def get_registration_options(user_id, username):
    """Generate WebAuthn registration options."""
    
    challenge = generate_challenge()
    # Store challenge for verification later
    session_store[user_id] = {
        'challenge': challenge,
        'type': 'registration',
        'expires': datetime.utcnow() + timedelta(minutes=5)
    }
    
    options = {
        'challenge': challenge,
        'rp': {
            'name': 'Your Application',
            'id': 'yourdomain.com'
        },
        'user': {
            'id': base64.urlsafe_b64encode(user_id.encode()).rstrip(b'='),
            'name': username,
            'displayName': username
        },
        'pubKeyCredParams': [
            {'type': 'public-key', 'alg': -7},   # ES256
            {'type': 'public-key', 'alg': -257}  # RS256
        ],
        'authenticatorSelection': {
            'authenticatorAttachment': 'platform',
            'requireResidentKey': True,
            'userVerification': 'preferred'
        },
        'timeout': 60000,
        'attestation': 'direct'
    }
    
    return json.dumps(options)
```

### Client: Create Credential

```javascript
async function registerCredential(options) {
  const publicKeyOptions = {
    ...JSON.parse(options),
    challenge: Uint8Array.from(atob(options.challenge), c => c.charCodeAt(0)),
    user: {
      ...options.user,
      id: Uint8Array.from(atob(options.user.id), c => c.charCodeAt(0))
    }
  };
  
  try {
    const credential = await navigator.credentials.create({
      publicKey: publicKeyOptions
    });
    
    return serializeCredential(credential);
  } catch (error) {
    console.error('Registration failed:', error);
    throw error;
  }
}

function serializeCredential(credential) {
  return {
    id: credential.id,
    rawId: Array.from(new Uint8Array(credential.rawId)),
    response: {
      attestationObject: Array.from(new Uint8Array(credential.response.attestationObject)),
      clientDataJSON: Array.from(new Uint8Array(credential.response.clientDataJSON))
    },
    type: credential.type
  };
}
```

### Server: Verify Registration

```python
import json
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.backends import default_backend

def verify_registration(user_id, credential_data):
    """Verify the registration credential."""
    
    stored = session_store.get(user_id)
    if not stored or stored['type'] != 'registration':
        raise ValueError("Invalid registration session")
    
    # Decode attestation object
    attestation_obj = credential_data['response']['attestationObject']
    # Verify signature, parse CBOR, extract public key
    # In production, use a library like py_webauthn
    
    # Extract public key and store securely
    public_key = parse_attestation(attestation_obj)
    
    # Store credential with user
    db.credentials.insert({
        'user_id': user_id,
        'credential_id': credential_data['id'],
        'public_key': public_key,
        'created': datetime.utcnow()
    })
    
    return True
```

## Authentication Implementation

Authentication (login) follows a similar pattern but verifies rather than creates credentials.

### Server: Generate Authentication Options

```python
def get_authentication_options(user_id):
    """Generate authentication options for user login."""
    
    challenge = generate_challenge()
    credentials = db.credentials.find({'user_id': user_id})
    
    session_store[f"auth_{user_id}"] = {
        'challenge': challenge,
        'type': 'authentication',
        'expires': datetime.utcnow() + timedelta(minutes=5)
    }
    
    options = {
        'challenge': challenge,
        'rpId': 'yourdomain.com',
        'allowCredentials': [
            {'id': cred['credential_id'], 'type': 'public-key'}
            for cred in credentials
        ],
        'userVerification': 'preferred',
        'timeout': 60000
    }
    
    return json.dumps(options)
```

### Client: Get Assertion

```javascript
async function authenticate(options) {
  const authOptions = {
    ...JSON.parse(options),
    challenge: Uint8Array.from(atob(options.challenge), c => c.charCodeAt(0)),
    allowCredentials: options.allowCredentials.map(cred => ({
      ...cred,
      id: Uint8Array.from(atob(cred.id), c => c.charCodeAt(0))
    }))
  };
  
  const assertion = await navigator.credentials.get({
    publicKey: authOptions
  });
  
  return serializeAssertion(assertion);
}
```

### Server: Verify Authentication

```python
def verify_authentication(user_id, assertion_data):
    """Verify the authentication assertion."""
    
    stored = session_store.get(f"auth_{user_id}")
    if not stored or stored['type'] != 'authentication':
        raise ValueError("Invalid authentication session")
    
    credential = db.credentials.find_one({
        'user_id': user_id,
        'credential_id': assertion_data['id']
    })
    
    if not credential:
        raise ValueError("Credential not found")
    
    # Verify signature using stored public key
    client_data = assertion_data['response']['clientDataJSON']
    signature = assertion_data['response']['signature']
    
    # Verify the signature
    verify_signature(
        credential['public_key'],
        stored['challenge'],
        client_data,
        signature
    )
    
    # Login successful - create session
    return create_session(user_id)
```

## Implementation Best Practices

### Relying Party ID Management

Your `rpId` (Relying Party ID) must match your domain exactly or be a parent domain. For `app.example.com`, you can use either `app.example.com` or `example.com`. Subdomains are treated as separate origins.

### Credential ID Storage

Store credential IDs as binary data, not base64 strings, in your database. This prevents encoding issues and allows efficient lookups.

### User Handle Implementation

For passwordless flows where users don't enter usernames, implement the user handle:

```javascript
// During registration, request a resident key
authenticatorSelection: {
  requireResidentKey: true,
  residentKey: 'required'
}
```

This allows the authenticator to store multiple credentials, enabling username-less authentication.

### Error Handling

WebAuthn operations can fail for various reasons:

```javascript
try {
  credential = await navigator.credentials.create({ publicKey: options });
} catch (error) {
  switch (error.name) {
    case 'InvalidStateError':
      // Authenticator already registered
      break;
    case 'NotAllowedError':
      // User cancelled or timeout
      break;
    case 'SecurityError':
      // HTTPS required or context invalid
      break;
    default:
      // Handle unknown errors
  }
}
```

## Security Considerations

Always serve WebAuthn pages over HTTPS. The WebAuthn API is only available in secure contexts. For local development, use `localhost` or configure self-signed certificates.

Implement rate limiting on registration and authentication endpoints to prevent abuse. Monitor for duplicate credential registrations from the same authenticator.

Store public keys with appropriate access controls. While public keys aren't secret, they should be protected against modification.

## Conclusion

WebAuthn provides strong, phishing-resistant authentication without passwords. The implementation requires careful attention to challenge management, signature verification, and credential storage. Start with a reputable library for your language—implementing crypto from scratch introduces security risks.

With proper implementation, you can offer users passwordless login via security keys, platform authenticators, or biometric devices, significantly improving your application's security posture.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
