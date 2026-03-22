---
layout: default
title: "Secure OAuth2 Implementation Checklist"
description: "A practical OAuth2 security checklist covering PKCE, token storage, redirect URI validation, scope minimization, and common implementation pitfalls to avoid"
date: 2026-03-22
author: theluckystrike
permalink: /secure-oauth2-implementation-checklist/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Secure OAuth2 Implementation Checklist

OAuth2 is widely implemented and widely misimplemented. Most OAuth2 vulnerabilities are not flaws in the spec — they are mistakes in how developers integrate it. This guide gives you a practical checklist with code examples for each control point.

## Before You Start: Flow Selection

Choosing the wrong OAuth2 flow is the first mistake. Use this decision tree:

```
Your application is:
├── Server-side web app (secret can be kept)      → Authorization Code + PKCE
├── Single-page app (no server secret)            → Authorization Code + PKCE
├── Mobile/native app                             → Authorization Code + PKCE
├── Trusted first-party with direct access        → Resource Owner Password (avoid if possible)
└── Machine-to-machine / background service       → Client Credentials
```

**Never use the Implicit flow.** It was deprecated in RFC 8252 for good reason — access tokens in URL fragments are visible in browser history, referrer headers, and server logs.

## Checklist

### 1 — Use PKCE for All Public Clients

PKCE (Proof Key for Code Exchange) prevents authorization code interception attacks. It's required for SPAs and mobile apps and recommended everywhere.

```javascript
// Node.js example — generating PKCE challenge
const crypto = require('crypto');

function generateCodeVerifier() {
  return crypto.randomBytes(32).toString('base64url');
}

function generateCodeChallenge(verifier) {
  return crypto
    .createHash('sha256')
    .update(verifier)
    .digest('base64url');
}

const verifier = generateCodeVerifier();
const challenge = generateCodeChallenge(verifier);

// Store verifier in session; send challenge in authorization request
const authUrl = new URL('https://auth.example.com/authorize');
authUrl.searchParams.set('response_type', 'code');
authUrl.searchParams.set('client_id', CLIENT_ID);
authUrl.searchParams.set('redirect_uri', REDIRECT_URI);
authUrl.searchParams.set('code_challenge', challenge);
authUrl.searchParams.set('code_challenge_method', 'S256');
authUrl.searchParams.set('state', generateState()); // see #2
authUrl.searchParams.set('scope', 'openid profile email');
```

In the token exchange, include the original verifier:

```javascript
const tokenResponse = await fetch('https://auth.example.com/token', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({
    grant_type: 'authorization_code',
    code: authorizationCode,
    redirect_uri: REDIRECT_URI,
    client_id: CLIENT_ID,
    code_verifier: verifier, // the verifier, not the challenge
  }),
});
```

**Checklist item**: `[ ] PKCE with S256 challenge method used on all authorization code requests`

### 2 — Validate State Parameter Against CSRF

The `state` parameter prevents CSRF attacks on the callback endpoint:

```javascript
// Before redirecting to authorization server
const state = crypto.randomBytes(16).toString('hex');
req.session.oauthState = state;

// On callback — verify state before processing
app.get('/callback', (req, res) => {
  const { code, state } = req.query;

  if (!state || state !== req.session.oauthState) {
    return res.status(400).send('Invalid state parameter');
  }

  delete req.session.oauthState;
  // proceed with token exchange
});
```

**Checklist item**: `[ ] State parameter generated cryptographically, validated on callback`

### 3 — Validate Redirect URIs Exactly

```javascript
// Authorization server side — STRICT comparison, no wildcards
const ALLOWED_REDIRECT_URIS = [
  'https://app.example.com/callback',
  'https://app.example.com/mobile-callback',
];

function validateRedirectUri(uri) {
  // Exact match — no substring matching, no wildcards
  if (!ALLOWED_REDIRECT_URIS.includes(uri)) {
    throw new Error('Invalid redirect_uri');
  }
  // Also validate the URI itself is well-formed
  const parsed = new URL(uri);
  if (parsed.protocol !== 'https:') {
    throw new Error('redirect_uri must use HTTPS');
  }
}
```

**Checklist items**:
```
[ ] Redirect URIs registered explicitly — no wildcards
[ ] Redirect URI validated via exact string match, not partial
[ ] HTTP redirect URIs blocked (HTTPS only) in production
```

### 4 — Token Storage

Where you store tokens determines your attack surface:

```javascript
// BAD: localStorage — accessible to XSS
localStorage.setItem('access_token', token); // DO NOT DO THIS

// BETTER: Memory only (lost on page refresh, but safest for SPAs)
let accessToken = null; // in-memory, never persisted

// BEST for server-side apps: HTTP-only cookies
res.cookie('access_token', token, {
  httpOnly: true,     // not accessible via document.cookie
  secure: true,       // HTTPS only
  sameSite: 'strict', // CSRF protection
  maxAge: 3600 * 1000, // 1 hour
  path: '/',
});
```

**Checklist items**:
```
[ ] Access tokens NOT stored in localStorage or sessionStorage
[ ] Tokens in HTTP-only, Secure, SameSite=Strict cookies OR memory
[ ] Refresh tokens stored server-side when possible
```

### 5 — Scope Minimization

Request only what you need:

```javascript
// BAD — requesting everything
scope: 'read write admin delete'

// GOOD — request minimum needed
scope: 'openid profile email'

// For elevated operations, request incrementally
function requestElevatedScope() {
  // Only request write scope when the user explicitly performs a write action
  initiateAuthFlow({ scope: 'openid profile email write:documents' });
}
```

**Checklist item**: `[ ] Scopes limited to minimum required for each use case`

### 6 — Token Expiry and Rotation

```javascript
// Short-lived access tokens (15 minutes to 1 hour)
// Refresh tokens with rotation

app.post('/token', async (req, res) => {
  const { grant_type, refresh_token } = req.body;

  if (grant_type === 'refresh_token') {
    const oldToken = await db.refreshTokens.findOne({ token: refresh_token });

    if (!oldToken || oldToken.used) {
      // Refresh token reuse detected — revoke all user tokens
      await db.refreshTokens.deleteMany({ userId: oldToken?.userId });
      return res.status(401).json({ error: 'invalid_grant' });
    }

    // Mark old token as used
    await db.refreshTokens.updateOne(
      { token: refresh_token },
      { $set: { used: true } }
    );

    // Issue new access token + new refresh token
    const newAccessToken = generateAccessToken(oldToken.userId);
    const newRefreshToken = generateRefreshToken(oldToken.userId);

    res.json({
      access_token: newAccessToken,
      refresh_token: newRefreshToken,
      expires_in: 900,
    });
  }
});
```

**Checklist items**:
```
[ ] Access token lifetime <= 1 hour
[ ] Refresh tokens rotated on each use
[ ] Refresh token reuse triggers revocation of all user tokens
[ ] Token revocation endpoint implemented
```

### 7 — ID Token Validation (OpenID Connect)

```javascript
const jwt = require('jsonwebtoken');
const jwksClient = require('jwks-rsa');

const client = jwksClient({
  jwksUri: 'https://auth.example.com/.well-known/jwks.json',
});

async function validateIdToken(idToken) {
  const decoded = jwt.decode(idToken, { complete: true });
  const key = await client.getSigningKey(decoded.header.kid);
  const publicKey = key.getPublicKey();

  const payload = jwt.verify(idToken, publicKey, {
    algorithms: ['RS256'],         // reject HS256 and 'none'
    audience: CLIENT_ID,           // validate aud claim
    issuer: 'https://auth.example.com', // validate iss claim
  });

  // Also check nonce if you sent one
  if (payload.nonce !== expectedNonce) {
    throw new Error('Nonce mismatch');
  }

  return payload;
}
```

**Checklist items**:
```
[ ] ID token signature verified against IdP's public keys
[ ] aud claim validated against your client_id
[ ] iss claim validated against expected issuer
[ ] algorithm restricted (never accept 'none' or HS256 with RS256 IdPs)
[ ] nonce validated if sent
```

## Full Security Checklist Summary

```
Authorization Code Flow
[ ] PKCE with S256 used for all public clients
[ ] State parameter validated to prevent CSRF
[ ] Redirect URIs registered and validated exactly

Token Handling
[ ] Access tokens not in localStorage
[ ] Refresh tokens rotated on use
[ ] Token expiry enforced (access < 1h, refresh < 30d)
[ ] Revocation endpoint implemented and used on logout

Client Configuration
[ ] Client secrets rotated periodically and stored in secrets manager
[ ] Scopes minimized per use case
[ ] Implicit flow not used anywhere

Server Validation
[ ] ID tokens validated (signature, aud, iss, nonce)
[ ] Authorization codes single-use
[ ] Codes expire within 10 minutes
```

## Related Reading

- [Secure GraphQL API Hardening Guide](/privacy-tools-guide/secure-graphql-api-hardening-guide/)
- [Secure CORS Configuration Guide](/privacy-tools-guide/secure-cors-configuration-guide/)
- [Secure WebRTC Implementation Guide](/privacy-tools-guide/secure-webrtc-implementation-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
