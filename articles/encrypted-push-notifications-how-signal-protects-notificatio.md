---
layout: default
title: "Encrypted Push Notifications: How Signal Protects Notification Content From Google and Apple"
description: "Learn how Signal implements encrypted push notifications to prevent Google and Apple from reading your message previews and metadata."
date: 2026-03-16
author: theluckystrike
permalink: /encrypted-push-notifications-how-signal-protects-notificatio/
---

Push notifications have become a ubiquitous part of mobile messaging, but they introduce a significant privacy concern: the notification payload passes through Google's Firebase Cloud Messaging (FCM) or Apple's APNs servers. This means platform operators can potentially read message sender information, notification text, and other metadata. Signal, the privacy-focused messaging app, addresses this vulnerability through a sophisticated encrypted push notification system that keeps sensitive content hidden from both platform operators and potential eavesdroppers.

## The Privacy Problem With Traditional Push Notifications

When you receive a traditional push notification, your device sends a device token to the messaging server. When a message arrives, the server sends a payload through FCM or APNs containing the sender ID, notification title, and message body. The push service then delivers this to your device.

This architecture means Google and Apple can see:
- The sender's identity
- The notification text (message previews)
- Message timestamps
- Delivery status
- Notification frequency patterns

For most users, this might seem acceptable. However, for users requiring strong privacy guarantees—journalists, activists, or anyone discussing sensitive matters—this metadata exposure represents a meaningful attack surface.

## Signal's Solution: Sealed Sender and Encrypted Payloads

Signal implements two complementary mechanisms to protect push notification content: Sealed Sender and encrypted payload delivery.

### Sealed Sender: Removing Sender Information

Traditional push notifications include the sender's identifier in the payload. Signal's Sealed Sender removes this information, allowing the server to deliver a notification without knowing who sent it.

Here's how the flow works conceptually:

```python
# Simplified conceptual flow for Sealed Sender
def create_sealed_notification(sender_public_key, recipient_device_token, message):
    # Generate one-time use keypair for this notification
    ephemeral_keypair = generate_ecdh_keypair()
    
    # Create encrypted envelope
    envelope = {
        "sender": encrypt(sender_public_key, ephemeral_keypair.private),
        "recipient": recipient_device_token,
        "timestamp": current_unix_timestamp()
    }
    
    # Server can deliver to recipient but cannot identify sender
    return push_service.deliver(envelope)
```

The key insight is that the push server acts as an opaque transport layer—it knows *where* to deliver the notification but not *who* sent it or potentially *what* it contains.

### Encrypted Payload Delivery

Sealed Sender alone doesn't protect the notification content. To solve this, Signal encrypts the notification payload itself. The device receives an opaque blob that only the Signal app can decrypt.

## Implementation Details: How Signal Encrypts Push Content

The actual implementation uses a multi-layer encryption approach:

```javascript
// Conceptual encrypted notification payload structure
const encryptedNotification = {
  // Layer 1: Encryption for push service transport
  encrypted_payload: base64(encrypt_aes_gcm(
    plaintext: JSON.stringify({
      message_preview: "...",  // Already encrypted
      sender_identity: "...",   // Blinded identifier
      timestamp: 1234567890
    }),
    key: push_service_key
  )),
  
  // Layer 2: Signal-specific encryption (device-to-device)
  signal_encrypted: base64(sealed_sender_encrypt(
    recipient_device: device_public_key,
    message: message_content
  ))
};
```

When your device receives this notification, the Signal app:
1. Decrypts the outer layer using keys stored only on your device
2. Processes the inner Signal-encrypted content
3. Displays a generic notification or the decrypted message preview (depending on your settings)

## Practical Implications for Developers

If you're building privacy-focused applications, Signal's approach offers several architectural lessons:

### Key Management

Signal uses a combination of identity keys and one-time ephemeral keys. The identity key provides consistent sender verification, while ephemeral keys prevent traffic analysis:

```python
# Pseudocode for notification encryption key derivation
def derive_notification_keys(identity_key, ephemeral_key, recipient_prekey):
    # ECDH key agreement
    shared_secret = ecdh(identity_key, ephemeral_key)
    
    # Derive separate keys for different purposes
    message_key = hkdf(shared_secret, "message-encryption")
    metadata_key = hkdf(shared_secret, "metadata-hiding")
    
    return {"message_key": message_key, "metadata_key": metadata_key}
```

### Blind Identifiers

To prevent the push service from correlating notifications with specific users, Signal uses cryptographic blinding:

```python
# Blinding the sender identifier from push service
def blind_sender_identifier(sender_identity_key, server_public_key):
    # The push server can verify the recipient exists
    # but cannot identify the specific sender
    blinded = sha256(ecdh(sender_identity_key, server_public_key))
    return blinded
```

### Notification Content Options

Signal provides users with granular control over notification content:

- **Show all content**: Full message preview in notification (highest convenience, lowest privacy)
- **No content**: Generic "New message" notification (highest privacy)
- **Contact-only**: Show previews only for contacts (balanced approach)

## Limitations and Considerations

While Signal's approach significantly improves privacy, some limitations exist:

1. **Metadata still flows**: While content is encrypted, connection metadata (timing, device identifiers) remains visible to platform operators
2. **Device token correlation**: If a device receives many notifications, pattern analysis could potentially identify users
3. **Fallback mechanisms**: When Sealed Sender isn't available (first message from new contact, offline devices), traditional notifications are used
4. **Battery and latency**: Additional encryption processing adds slight overhead

## Enabling Signal's Privacy Features

For Signal users wanting maximum protection:

1. Open Signal Settings → Notifications
2. Enable "Sealed Sender" (may show as "Allow from contacts" or "Always")
3. Choose notification display preference under "Show"

Note that both sender and recipient must have Sealed Sender enabled for full protection.

## Conclusion

Signal's encrypted push notification system demonstrates that practical privacy is achievable even when using platform-dependent notification services. By encrypting notification content and removing sender identifiers from the push payload, Signal ensures that Google and Apple see only opaque encrypted blobs rather than readable message content or sender identities.

For developers building privacy-conscious applications, this approach provides a template: separate transport concerns from message content, use cryptographic blinding for identifiers, and give users meaningful control over their privacy-utility tradeoff.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
