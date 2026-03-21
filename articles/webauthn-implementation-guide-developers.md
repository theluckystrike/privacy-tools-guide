---
layout: default
title: "WebAuthn Implementation Guide for Developers"
description: "A practical developer guide to implementing WebAuthn authentication. Code examples for registration, authentication, and best practices"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "theluckystrike"
permalink: /webauthn-implementation-guide-developers/
reviewed: true
score: 8
categories: [guides]
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}To implement WebAuthn, generate a cryptographic challenge on your server, call `navigator.credentials.create()` for registration or `navigator.credentials.get()` for authentication on the client, then verify the signed response server-side against the stored public key. This guide provides copy-ready Python and JavaScript code for the full registration and authentication flow, plus best practices for challenge management, credential storage, and error handling.

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
 {'type': 'public-key', 'alg': -7}, # ES256
 {'type': 'public-key', 'alg': -257} # RS256
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

## Using py_webauthn for Production Verification

The verification snippets above show the conceptual flow. For production code, use the `py_webauthn` library rather than implementing CBOR parsing and signature verification yourself:

```python
from webauthn import (
 generate_registration_options,
 verify_registration_response,
 generate_authentication_options,
 verify_authentication_response,
)
from webauthn.helpers.structs import (
 PublicKeyCredentialDescriptor,
 UserVerificationRequirement,
)

# Registration options
options = generate_registration_options(
 rp_id="yourdomain.com",
 rp_name="Your Application",
 user_id=user_id.encode(),
 user_name=username,
 user_display_name=display_name,
)

# Verify registration response from client
verification = verify_registration_response(
 credential=credential_response, # JSON from client
 expected_challenge=stored_challenge,
 expected_rp_id="yourdomain.com",
 expected_origin="https://yourdomain.com",
)

# Store the returned credential
db.credentials.insert({
 "user_id": user_id,
 "credential_id": verification.credential_id,
 "public_key": verification.credential_public_key,
 "sign_count": verification.sign_count,
 "created_at": datetime.utcnow().isoformat(),
})
```

The library handles CBOR decoding, attestation verification, and public key parsing. The `sign_count` field in the verification result is important: increment it with each authentication and reject requests where the count does not increase — this detects cloned authenticators.

## Supporting Multiple Authenticators Per User

Users should be able to register multiple devices (laptop, phone, hardware security key) for account recovery. Design your credential storage to support this from the start:

```python
# Schema that supports multiple credentials per user
CREATE TABLE webauthn_credentials (
 id SERIAL PRIMARY KEY,
 user_id UUID NOT NULL REFERENCES users(id),
 credential_id BYTEA NOT NULL UNIQUE,
 public_key BYTEA NOT NULL,
 sign_count INTEGER NOT NULL DEFAULT 0,
 device_name TEXT, -- User-assigned label: "Work YubiKey"
 created_at TIMESTAMPTZ DEFAULT NOW(),
 last_used TIMESTAMPTZ
);
```

Expose a credential management UI where users can:
- See all registered authenticators with their `device_name` and `last_used` date
- Rename devices for clarity
- Delete a lost or stolen device

When a device is deleted, generate a new challenge immediately on the server side to invalidate any in-progress authentication from that credential.

## Adding WebAuthn to an Existing Password-Based System

Most teams add WebAuthn as a second factor before removing passwords entirely. The migration path:

1. **Add as 2FA**: Users register a passkey after password login. Password remains required, passkey provides the second factor.
2. **Offer as alternative login**: Users who have registered a passkey can use it in place of the password + OTP flow.
3. **Deprecate passwords**: After majority passkey adoption, encourage password removal. Keep a recovery path (email link or support contact) for edge cases.

```javascript
// Login flow that supports both paths
async function login(username, password) {
 const user = await authenticateWithPassword(username, password);
 if (!user) throw new Error("Invalid credentials");

 const credentials = await getCredentials(user.id);

 if (credentials.length > 0) {
 // Upgrade: use passkey for second factor
 const authOptions = await getAuthenticationOptions(user.id);
 const assertion = await authenticate(authOptions);
 await verifyAuthenticationOnServer(assertion);
 }

 return createSession(user.id);
}
```

This progressive approach lets you introduce WebAuthn without requiring all users to migrate simultaneously, reducing support burden during the transition.


## Related Articles

- [Passkeys vs Passwords: Security Comparison FIDO2 WebAuthn](/privacy-tools-guide/passkeys-vs-passwords-security-comparison-fido2-webauthn-guide/)
- [WebAuthn vs FIDO2 vs Passkeys: Key Differences Explained](/privacy-tools-guide/webauthn-vs-fido2-vs-passkey-differences/)
- [Arti Tor Rust Implementation Explained 2026](/privacy-tools-guide/arti-tor-rust-implementation-explained-2026/)
- [Enterprise Privacy by Design Framework Implementation.](/privacy-tools-guide/enterprise-privacy-by-design-framework-implementation-guide-/)
- [GDPR Article 17 Erasure Implementation Code](/privacy-tools-guide/gdpr-article-17-erasure-implementation-code/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
