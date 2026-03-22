---
layout: default
title: "Secure JWT Implementation Best Practices"
description: "Avoid the most dangerous JWT vulnerabilities including algorithm confusion, weak secrets, and missing validation with practical code examples in Python and Node"
date: 2026-03-22
author: theluckystrike
permalink: /secure-jwt-implementation-best-practices/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
# Secure JWT Implementation Best Practices

JSON Web Tokens are everywhere — and so are JWT vulnerabilities. The `alg: none` bypass, HMAC/RSA confusion, weak shared secrets, and missing expiry checks have each led to serious auth bypasses in production systems. This guide covers the full set of secure implementation requirements with working code.

## JWT Structure Refresher

A JWT is three base64url-encoded parts joined by dots: `header.payload.signature`. The header declares the algorithm, the payload holds claims, and the signature proves integrity.

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9   <- header
.eyJzdWIiOiJ1c2VyXzEyMyIsImV4cCI6MTc0MzEwMDAwMH0  <- payload
.signature_bytes_here                              <- signature
```

**JWT does not encrypt by default.** Anyone who has the token can decode the header and payload. If you need confidentiality, use JWE (JSON Web Encryption), not plain JWT.

---

## Critical Vulnerabilities and Fixes

### 1. Algorithm Confusion (RS256 → HS256)

If your server accepts RS256 (asymmetric), an attacker can sometimes forge tokens by switching the algorithm to HS256 (symmetric) and signing with the public key as the HMAC secret — because the public key is often known.

**The fix**: always specify the expected algorithm explicitly on the server side:

```python
# INSECURE — trusts the 'alg' header from the token
import jwt
payload = jwt.decode(token, public_key, algorithms=None)  # NEVER do this

# SECURE — hard-code the expected algorithm
payload = jwt.decode(token, public_key, algorithms=["RS256"])
```

```javascript
// Node.js — jsonwebtoken library
// INSECURE
const payload = jwt.verify(token, publicKey);

// SECURE
const payload = jwt.verify(token, publicKey, { algorithms: ['RS256'] });
```

### 2. The `alg: none` Attack

Some libraries historically accepted tokens with `"alg": "none"` and no signature. Fix: use a library that rejects `none` by default and never list it in allowed algorithms.

```python
# Explicit rejection
ALLOWED_ALGORITHMS = ["RS256", "RS384", "RS512", "ES256", "ES384"]
# Never include "none", "HS256" if you use asymmetric keys
payload = jwt.decode(token, public_key, algorithms=ALLOWED_ALGORITHMS)
```

### 3. Weak HMAC Secrets

HS256 with a short or guessable secret can be brute-forced offline using tools like `hashcat`. The JWT signature is a MAC — it's only as strong as the secret.

```bash
# Attack: hashcat can try millions of candidates per second
hashcat -a 0 -m 16500 captured_jwt.txt wordlist.txt

# Generate a cryptographically strong secret (32 bytes minimum for HS256)
openssl rand -base64 64   # 48 bytes → 64 base64 chars
```

For production, prefer RS256 with a 2048+ bit RSA key or ES256 with P-256. HMAC is fine for internal services where you control both ends and the secret is stored in a secrets manager.

---

## Secure Token Generation

### Python (PyJWT)

```python
import jwt
import time
from pathlib import Path

# Load RS256 private key (generated with: openssl genrsa -out private.pem 2048)
PRIVATE_KEY = Path("/run/secrets/jwt_private_key.pem").read_text()

def create_token(user_id: str, roles: list[str]) -> str:
    now = int(time.time())
    payload = {
        "iss": "https://auth.example.com",     # issuer — who created the token
        "sub": str(user_id),                   # subject — the user
        "aud": "https://api.example.com",      # audience — intended recipient
        "iat": now,                            # issued at
        "nbf": now,                            # not valid before
        "exp": now + 3600,                     # expires in 1 hour
        "jti": secrets.token_hex(16),          # JWT ID — unique per token
        "roles": roles,
    }
    return jwt.encode(payload, PRIVATE_KEY, algorithm="RS256")
```

### Node.js (jsonwebtoken)

```javascript
const jwt = require('jsonwebtoken');
const fs  = require('fs');
const crypto = require('crypto');

const privateKey = fs.readFileSync('/run/secrets/jwt_private_key.pem');

function createToken(userId, roles) {
  return jwt.sign(
    {
      sub:   String(userId),
      aud:   'https://api.example.com',
      roles: roles,
      jti:   crypto.randomBytes(16).toString('hex'),
    },
    privateKey,
    {
      algorithm: 'RS256',
      issuer:    'https://auth.example.com',
      expiresIn: '1h',
      notBefore: 0,
    }
  );
}
```

---

## Secure Token Validation

Always validate **all** of these claims on every request:

```python
import jwt
from pathlib import Path

PUBLIC_KEY = Path("/run/secrets/jwt_public_key.pem").read_text()

def validate_token(token: str) -> dict:
    try:
        payload = jwt.decode(
            token,
            PUBLIC_KEY,
            algorithms=["RS256"],           # hard-coded — never dynamic
            audience="https://api.example.com",   # must match aud claim
            issuer="https://auth.example.com",    # must match iss claim
            options={
                "verify_exp": True,         # reject expired tokens
                "verify_nbf": True,         # reject tokens not yet valid
                "verify_iat": True,         # reject tokens with future iat
                "require": ["sub", "exp", "iat", "jti"],  # required claims
            },
            leeway=10,  # 10-second clock skew tolerance max
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise AuthError("Token has expired")
    except jwt.InvalidAudienceError:
        raise AuthError("Token not intended for this service")
    except jwt.InvalidIssuerError:
        raise AuthError("Unknown token issuer")
    except jwt.DecodeError as e:
        raise AuthError(f"Invalid token: {e}")
```

---

## Token Revocation

JWTs are stateless — they remain valid until expiry even if the user logs out or is banned. Fix this with a short expiry plus a blocklist for critical tokens:

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, db=0)

def revoke_token(jti: str, ttl_seconds: int):
    """Add JTI to blocklist with TTL matching token expiry."""
    r.setex(f"revoked_jti:{jti}", ttl_seconds, "1")

def is_revoked(jti: str) -> bool:
    return r.exists(f"revoked_jti:{jti}") > 0

# In validate_token, after decoding:
if is_revoked(payload["jti"]):
    raise AuthError("Token has been revoked")
```

Keep token TTL short (15–60 minutes for access tokens) and use refresh tokens with longer expiry for UX. Revoke refresh tokens on logout.

---

## Key Rotation

```bash
# Generate a new RSA key pair
openssl genrsa -out new_private.pem 2048
openssl rsa -in new_private.pem -pubout -out new_public.pem

# Publish the new public key in your JWKS endpoint
# /well-known/jwks.json should expose ALL currently valid public keys
# so tokens signed with the old key continue to validate during rollover
```

Implement a JWKS endpoint so API consumers can auto-fetch valid public keys:

```python
from fastapi import FastAPI
from jwt.algorithms import RSAAlgorithm
import json

app = FastAPI()

@app.get("/.well-known/jwks.json")
def jwks():
    public_key_pem = Path("/run/secrets/jwt_public_key.pem").read_text()
    jwk = json.loads(RSAAlgorithm.to_jwk(
        RSAAlgorithm.from_jwk(public_key_pem)
    ))
    jwk["kid"] = "2026-03-v1"   # key ID — increment on rotation
    jwk["use"] = "sig"
    return {"keys": [jwk]}
```

---

## Common Mistakes Reference

| Mistake | Risk | Fix |
|---|---|---|
| `algorithms=None` or `algorithms=[]` | Algorithm confusion / none bypass | Hard-code `algorithms=["RS256"]` |
| Missing `exp` validation | Tokens never expire | Always `verify_exp: True` |
| Missing `aud` validation | Token accepted by wrong service | Always verify `aud` |
| Short HS256 secret | Offline brute-force | Use RSA/EC or 256-bit secret |
| JWT in localStorage | XSS token theft | Use httpOnly secure cookies |
| Long TTL (days/weeks) | Stolen token long window | Access token ≤1h, refresh ≤14d |
| No JTI tracking | Can't revoke specific tokens | Store JTI in Redis on issue |

---

## Related Reading

- [Secure API Gateway Setup with Kong](/privacy-tools-guide/kong-api-gateway-secure-setup-guide/)
- [Secure Webhook Implementation Guide](/privacy-tools-guide/secure-webhook-implementation-guide/)
- [Secure Environment Variable Management](/privacy-tools-guide/secure-environment-variable-management-guide/)
- [Android Privacy Best Practices 2026](/privacy-tools-guide/android-privacy-best-practices-2026/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
