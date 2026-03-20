---

layout: default
title: "WebAuthn vs FIDO2 vs Passkeys: Key Differences Explained"
description: "A technical breakdown of WebAuthn, FIDO2, and passkeys for developers. Understand the standards, protocols, and implementations driving passwordless."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /webauthn-vs-fido2-vs-passkey-differences/
reviewed: true
score: 8
categories: [comparisons]
intent-checked: true
voice-checked: true
---


WebAuthn is the browser API for passwordless login, FIDO2 is the full protocol suite (WebAuthn plus the CTAP standard for communicating with hardware authenticators), and passkeys are the consumer-facing product built on FIDO2 that adds cross-device sync through iCloud Keychain, Google Password Manager, or Windows Hello. If you are a developer implementing passwordless auth, you work with the WebAuthn API; FIDO2 defines the underlying cryptographic protocol; and passkeys are what your end users actually interact with.

## What Is WebAuthn?

WebAuthn is a web API standard developed by the World Wide Web Consortium (W3C) that enables web applications to use public-key cryptography for authentication. Launched in 2018, WebAuthn provides a standardized JavaScript API that browsers expose to web applications.

The WebAuthn API allows servers to register and authenticate users using authenticators—devices like security keys, smartphones, or platform-specific biometric systems. When a user registers, the authenticator generates a public-private key pair. The private key remains on the device, while the public key gets sent to the server.

A basic WebAuthn registration looks like this in JavaScript:

```javascript
const publicKeyCredentialCreationOptions = {
  challenge: new Uint8Array([/* server-provided challenge */]),
  rp: {
    name: "Your Application",
    id: "yourdomain.com"
  },
  user: {
    id: new Uint8Array([/* user ID bytes */]),
    name: "username",
    displayName: "User Name"
  },
  pubKeyCredParams: [
    { type: "public-key", alg: -7 },  // ES256
    { type: "public-key", alg: -257 } // RS256
  ],
  authenticatorSelection: {
    authenticatorAttachment: "platform",
    requireResidentKey: true,
    userVerification: "preferred"
  }
};

const credential = await navigator.credentials.create({
  publicKey: publicKeyCredentialCreationOptions
});
```

This code creates a credential on a platform authenticator (like Windows Hello or Touch ID). The returned credential object contains the public key that your server stores for future authentication attempts.

## What Is FIDO2?

FIDO2 is a set of specifications published by the FIDO Alliance that enables passwordless authentication. It consists of two components: the Client-to-Authenticator Protocol (CTAP) and WebAuthn.

CTAP defines how clients (browsers, operating systems) communicate with external authenticators. CTAP1 (or U2F) was the original FIDO standard using only attestation certificates. CTAP2 introduced resident keys, allowing authenticators to store multiple credentials.

The FIDO2 architecture has three parties:

1. **Relying Party** - The web server that wants to authenticate users
2. **Client** - The browser or platform that implements WebAuthn
3. **Authenticator** - The device that creates and stores cryptographic keys

FIDO2 requires that authenticators use hardware-backed key storage. This means the private keys never leave the authenticator in a form that any other device can access. Even if a server is compromised, attackers cannot steal usable credentials because they would need physical access to the user's authenticator.

FIDO2 also supports attestation—cryptographic statements that verify the authenticator's make and model. This lets servers make informed decisions about which authenticators they accept:

```javascript
// Server-side verification of attestation
const attestationStatement = credential.response.attestationStatement;
const attestationTrustPath = verifyAttestation(attestationStatement, clientDataHash);

// Different relying parties may accept different authenticator types
const acceptedAttestationFormats = ['packed', 'fido-u2f', 'none'];
```

## What Are Passkeys?

Passkeys are a consumer-facing implementation of FIDO2 credentials, promoted by Apple, Google, and Microsoft through the FIDO Alliance. While FIDO2 and WebAuthn are technical standards, passkeys represent how those standards reach end users.

When you create a passkey on your iPhone, it syncs through iCloud Keychain. On Android, it syncs through Google Password Manager. This sync capability is not part of the FIDO2 specification—it is an extension added by the major platform vendors.

Passkeys solve a key problem with hardware security keys: portability. With traditional FIDO2 authenticators, if you lose your YubiKey, you lose access to your accounts. Passkeys sync across devices, providing the same phishing resistance without the single-point-of-failure risk.

Three features distinguish passkeys from generic FIDO2 credentials:

1. **Cross-device sync** - Credentials flow between your devices automatically
2. **Discoverable credentials** - The authenticator can find the right credential without the server providing a credential ID
3. **Platform integration** - Operating systems present native UI for passkey creation and use

## How They Connect

The relationship is straightforward: WebAuthn is the web API, FIDO2 is the protocol suite (including CTAP), and passkeys are the consumer product built on top of FIDO2.

When you use a passkey on a website, the following happens:

1. The website calls the WebAuthn API (`navigator.credentials.get()`)
2. The browser communicates with the platform's passkey provider (iCloud Keychain, Google Password Manager, Windows Hello)
3. The provider uses FIDO2 CTAP commands to access the credential
4. The authenticator signs an authentication assertion using the stored private key
5. The server verifies the signature using the registered public key

This chain of standards means any passkey implementation works with any WebAuthn-enabled website. A passkey created on iOS will work on Chrome for Linux, provided both platforms support the relevant specifications.

## Practical Considerations for Developers

Implementing WebAuthn requires server-side changes beyond simple password authentication. Your server must:

- Generate and store challenges (cryptographic nonces)
- Verify attestation statements if requiring specific authenticator types
- Handle credential IDs and associate them with user accounts
- Implement account recovery mechanisms

For most developers, using a library like SimpleWebAuthn or Authelia saves significant implementation time:

```javascript
// Server-side with SimpleWebAuthn
const { verifyRegistrationResponse } = require('@simplewebauthn/server');

const verification = await verifyRegistrationResponse({
  response: credential,
  expectedChallenge: savedChallenge,
  expectedOrigin: 'https://yourdomain.com',
  expectedRPID: 'yourdomain.com'
});

if (verification.verified) {
  // Save credential.publicKey and credential.id to your database
  await saveCredential(userId, verification.registrationInfo);
}
```

The key decision point is whether to require attestation. Attestation lets you enforce hardware security key requirements, but it complicates implementation and may frustrate users who just want to use their phone.

## Summary

WebAuthn provides the browser API, FIDO2 defines the protocol, and passkeys deliver the user experience. For authentication, WebAuthn and FIDO2 are synonymous at the technical level—WebAuthn is the web-facing part, FIDO2 encompasses both web and native authenticator communication.

Passkeys add synchronization and discoverability on top of FIDO2, making credentials practical for everyday use. The standards are mature and browser support is excellent across Chrome, Firefox, Safari, and Edge. If you're building a new application, WebAuthn with passkey support represents the strongest authentication path available today.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [AI Tools Compared](/ai-tools-compared/){: .cross-repo-linked}
- More guides coming soon.
- [Passkey Adoption Timeline by Platform: A Developer Guide](/privacy-tools-guide/passkey-adoption-timeline-by-platform/)
- [How to Use Password Manager with YubiKey Hardware Key Setup](/privacy-tools-guide/how-to-use-password-manager-with-yubikey-hardware-key-setup/)
- [TOTP vs FIDO2 Authentication Explained: A Developer's Guide](/privacy-tools-guide/totp-vs-fido2-authentication-explained/)

Built by