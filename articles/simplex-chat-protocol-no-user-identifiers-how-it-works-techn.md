---
layout: default
title: "Simplex Chat Protocol No User Identifiers How It Works Techn"
description: "A technical deep dive into how SimpleX Chat achieves privacy through absence of user identifiers. Learn the queue-based architecture, DH key exchange"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /simplex-chat-protocol-no-user-identifiers-how-it-works-techn/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

The SimpleX Chat protocol represents a fundamental shift in how secure messaging systems handle user identity. Unlike Signal, which uses phone numbers, or Session, which uses blockchain-based identifiers, SimpleX eliminates user identifiers entirely from its core architecture. This design decision creates a messaging system where intercepting traffic reveals nothing about who owns or operates the participating clients.

## The Core Problem with Traditional Messaging

Most encrypted messaging protocols require some form of user identifier to establish connections. When Alice wants to message Bob, her client needs Bob's identifier—typically a phone number, username, or public key fingerprint—to initiate the handshake. This creates several privacy vulnerabilities:

- **Correlation attacks**: Metadata linking identifiers to communication patterns
- **Directory attacks**: Centralized services storing who talks to whom
- **Social graph reconstruction**: Easy mapping of relationships

SimpleX addresses these by removing the permanent identifier entirely. Instead, the protocol uses ephemeral queue identifiers and directory servers that store only temporary connection data.

## Queue-Based Architecture

The SimpleX protocol organizes communication around message queues rather than user accounts. Each queue serves as an one-way communication channel between two participants. For bidirectional messaging, you need two queues—one for each direction.

### Queue Types

SimpleX implements three queue types:

1. **Contact Queues**: Permanent queues for ongoing conversations
2. **Group Queues**: Shared queues for group communication
3. **Anonymous Queues**: Ephemeral queues for unknown contacts

When you first connect to SimpleX, your client creates a unique queue identity. This queue identifier differs from a user identifier—it's tied to a specific device installation, not a user account.

### Queue Structure

Each queue contains:

```json
{
  "queue_id": "q_abc123def456",
  "public_key": "b64_encoded_key...",
  "created_at": 1706918400,
  "sender_ratchet_key": "b64_encoded..."
}
```

The queue ID serves only for initial connection establishment. Once connected, clients exchange ratchet keys that enable continuous key rotation without referencing the original queue ID.

## The Connection Process Without Identifiers

Here's how a typical connection establishes without either party knowing the other's permanent identity:

### Step 1: Creating an Invitation

Alice generates an invitation link containing her queue ID and initial public key:

```python
# Simplified Python representation of invitation creation
def create_invitation(client_keypair):
    invitation = {
        "queue_id": generate_queue_id(),
        "public_key": client_keypair.public_key,
        "ratchet_key": generate_ratchet_keypair().public_key,
        "signature": sign(client_keypair.private_key,
                          queue_id + public_key + ratchet_key)
    }
    return encode_invitation(invitation)
```

This invitation link goes to Bob through any out-of-band channel—Signal, email, or carrier pigeon. The link contains no information about Alice's identity, only connection parameters.

### Step 2: Initial Handshake

When Bob receives the invitation, his client performs a Diffie-Hellman key exchange:

```python
def process_invitation(invitation):
    # Generate ephemeral keypair for DH
    bob_ephemeral = generate_keypair()

    # Perform DH with Alice's keys
    shared_secret = dh(
        bob_ephemeral.private_key,
        invitation['public_key']
    )

    # Derive initial symmetric key
    symmetric_key = kdf(shared_secret,
                       context="simpleX-v1")

    # Store established key
    return {
        "session_key": symmetric_key,
        "alice_queue_id": invitation['queue_id']
    }
```

The critical aspect: Alice's queue ID exists only for routing messages through the directory server. Once the handshake completes, all further messages use ratchet-derived keys that contain no reference to either participant's identity.

### Step 3: Queue Address Rotation

After establishing the initial connection, both clients immediately rotate to new queue addresses:

```python
def rotate_queue(session):
    new_queue = generate_new_queue()
    new_ratchet_key = generate_ratchet_keypair()

    # Encrypt new address using existing session key
    encrypted_address = encrypt(
        new_queue.address + new_ratchet_key.public_key,
        session.session_key
    )

    # Send rotation message through old queue
    send_message(session, {
        "type": "QUEUE_ROTATION",
        "payload": encrypted_address
    })

    return new_queue, new_ratchet_key
```

This rotation ensures that even if someone monitors the initial connection, they cannot correlate future messages with the original queue.

## Message Delivery Architecture

SimpleX separates message transmission from identity verification:

1. **SMP Server**: Handles message queuing and delivery (SimpleX Message Broker)
2. **Client**: Performs all encryption, identity verification, and message storage

The SMP server sees only encrypted blobs with queue IDs. It cannot determine:
- Who owns each queue
- Who sends messages to whom
- What the messages contain

### Server Interaction Protocol

Clients interact with SMP servers using this basic flow:

```python
def send_message_via_smp(server, queue_id, encrypted_payload):
    # Server stores message associated only with queue_id
    server.queue_message(
        queue_id=queue_id,
        message=encrypted_payload,
        timestamp=current_time()
    )

    # Recipient polls for new messages
    messages = server.get_messages(queue_id=recipient_queue)

    # Only recipient can decrypt messages for their queue
    for msg in messages:
        decrypted = decrypt(msg, recipient_session_key)
```

The server enforces no identity requirements—any client with a valid queue ID can post messages to it.

## Double Ratchet Implementation

SimpleX implements the Double Ratchet algorithm with some modifications for its identifier-free design:

```python
class SimpleXRatchet:
    def __init__(self, shared_secret):
        self.root_key = shared_secret
        self.chain_key = hash(shared_secret)

    def ratchet_step(self, peer_ratchet_key):
        # DH ratchet - derive new chain from peer's ratchet key
        new_shared = dh(self.private_key, peer_ratchet_key)
        self.root_key = kdf(self.root_key + new_shared, "ratchet")
        self.chain_key = hash(self.root_key)

    def encrypt_message(self, plaintext):
        # Symmetric ratchet - advance chain for each message
        self.chain_key, message_key = derive_keys(self.chain_key)
        return encrypt(plaintext, message_key)
```

This ensures forward secrecy—even if long-term keys compromise, past messages remain secure due to the ratchet progression.

## Practical Privacy Implications

This architecture provides several concrete privacy properties:

**No Social Graph Metadata**: Directory servers never learn who communicates with whom because queue IDs change and carry no user binding.

**Ephemeral Connections**: Invitation links work once. After use, they're worthless for future correlation.

**Deniable Authentication**: Because no permanent keys exist, messages cannot be cryptographically proven to originate from a specific user—providing plausible deniability.

**Server Blindness**: SMP servers store only encrypted blobs. Compromise reveals message contents but no sender/recipient identities.

## Implementation Considerations

For developers implementing SimpleX or similar identifier-free protocols:

1. **Queue Management**: Implement queue rotation on every session initialization
2. **Key Storage**: Use secure enclaves or hardware-backed storage for long-term ratchet keys
3. **Server Selection**: Allow users to run their own SMP servers for complete infrastructure control
4. **Forward Secrecy**: Rotate symmetric keys frequently—SimpleX recommends per-message rotation

The identifier-free design trades some usability for privacy. Users must share new invitations for each device, and there's no way to recover access if all queue credentials lost. These tradeoffs reflect the protocol's priority: privacy over convenience.

---


## Related Articles

- [Simplex Chat Review: No Identifiers Architecture Analysis](/privacy-tools-guide/simplex-chat-review-no-identifiers/)
- [Signal vs Session vs SimpleX](/privacy-tools-guide/signal-vs-session-vs-simplex-secure-messaging-comparison/)
- [Configure Xray Reality Protocol for Undetectable Proxy from](/privacy-tools-guide/how-to-configure-xray-reality-protocol-for-undetectable-prox/)
- [Mimblewimble Protocol Privacy Features How Grin And Beam Pro](/privacy-tools-guide/mimblewimble-protocol-privacy-features-how-grin-and-beam-pro/)
- [Mls Messaging Layer Security Protocol How It Will Change](/privacy-tools-guide/mls-messaging-layer-security-protocol-how-it-will-change-group-encryption-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
