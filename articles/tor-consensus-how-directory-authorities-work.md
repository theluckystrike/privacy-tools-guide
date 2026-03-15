---
layout: default
title: "Tor Consensus: How Directory Authorities Work"
description: "A technical deep-dive into Tor network consensus mechanism and directory authorities. Learn how relay information is validated, how clients bootstrap securely, and how to interact with the consensus programmatically."
date: 2026-03-15
author: theluckystrike
permalink: /tor-consensus-how-directory-authorities-work/
categories: [privacy, security, tor, networking]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

The Tor network relies on a decentralized system of relays to provide anonymity, but clients need a reliable way to discover which relays exist and which ones to trust. This is where the **consensus** mechanism and **directory authorities** come into play. Understanding this infrastructure is essential for developers building Tor-integrated applications, security researchers auditing network integrity, or power users who want to verify their connection's reliability.

## What Are Directory Authorities

Directory authorities (DAs) are specially designated servers that form the backbone of Tor's directory system. There are nine primary authorities operated by different entities around the world—each managed by trusted organizations or individuals committed to the Tor project's mission.

These authorities perform three critical functions:

1. **Relay voting** — Each authority collects relay descriptors from all published relays
2. **Consensus generation** — Authorities vote on the network state and produce a unified document
3. **Document signing** — The consensus is signed by a majority of authorities, creating a verifiable snapshot

The current list of directory authorities includes well-known nodes like `moria1`, `tor26`, `dizum`, and others. You can retrieve this list from any Tor relay's descriptor or from the Tor project directly.

## The Consensus Mechanism Explained

The Tor consensus is a single, signed document that lists every relay in the network along with its metadata: fingerprint, exit policy, bandwidth, flags, and cryptographic identity. Clients download this document typically once per hour (the consensus interval) and use it to make routing decisions.

Here's how the process works:

**Step 1: Relay Publication**
Every Tor relay periodically publishes a **server descriptor** containing its identity key, exit policy, bandwidth, and other configuration details. Relays also publish **extra-info documents** containing additional statistics.

**Step 2: Authority Collection**
Directory authorities continuously fetch these descriptors from all known relays. Each authority builds its own view of the network based on what it has collected.

**Step 3: Voting and Consensus**
Once per hour, each authority publishes its **vote**—a signed document listing all relays the authority believes are valid. Authorities exchange votes with each other and compute a **consensus** by applying a weighted voting algorithm. Relays must meet certain criteria (uptime, bandwidth, proper configuration) to receive flags like "Fast", "Stable", or "Guard".

**Step 4: Distribution**
The final consensus document is signed by at least 5 of 9 authorities (a quorum requirement). This signature threshold ensures that no single compromised authority can manipulate the network state. Clients and relays download the signed consensus and verify the signatures before accepting it.

## Practical Examples for Developers

### Fetching the Current Consensus

You can retrieve the current consensus directly from a directory mirror or authority:

```bash
curl -s https://onionoo.torproject.org/details | jq '.relays[] | select(.flags | contains(["Fast", "Stable", "Guard"])) | .nickname, .fingerprint, .bandwidth'
```

The Onionoo protocol provides a RESTful API for querying Tor network status. For the raw consensus, you can hit:

```bash
curl -s https://consensus.torproject.org/consensus/all | head -100
```

### Parsing the Consensus in Python

If you're building tools that need to consume the consensus, here's how you might parse it:

```python
import stem.descriptor.remote

def get_active_guards():
    """Fetch consensus and return available guard relays."""
    consensus = stem.descriptor.remote.get_consensus(
        endpoints=[
            '93.95.228.37:80',  # tor26
            '199.254.238.52:80',  # moria1
        ]
    ).run()[0]
    
    guards = []
    for relay in consensus:
        if 'Guard' in relay.flags and 'Stable' in relay.flags:
            guards.append({
                'nickname': relay.nickname,
                'fingerprint': relay.fingerprint,
                'or_port': relay.or_port,
                'bandwidth': relay.bandwidth,
            })
    
    return guards

if __name__ == '__main__':
    for guard in get_active_guards():
        print(f"{guard['nickname']}: {guard['bandwidth']} KB/s")
```

This uses Stem, the Python controller library for Tor, to fetch the consensus and filter for stable guard relays.

### Verifying Consensus Signatures

For security-sensitive applications, you should verify that the consensus meets the quorum threshold:

```python
from stem.descriptor import Consensus

def verify_consensus(consensus_path):
    """Verify consensus has sufficient authority signatures."""
    with open(consensus_path, 'rb') as f:
        consensus = Consensus(f.read())
    
    valid_signatures = 0
    required = 5  # Quorum threshold
    
    for sig in consensus.signatures:
        if sig.is_valid:
            valid_signatures += 1
    
    if valid_signatures < required:
        raise ValueError(f"Insufficient signatures: {valid_signatures}/{required}")
    
    return True
```

## Understanding Relay Flags

The consensus assigns various flags to relays based on their performance and characteristics. These flags directly influence which relays your Tor client selects:

- **Fast**: Relay with bandwidth >= 100 KB/s average
- **Stable**: Relay with >= 7 days uptime
- **Guard**: Relay suitable for use as an entry guard (high bandwidth, stable, long uptime)
- **Exit**: Relay that allows exit connections
- **V2Dir**: Relay that serves directory documents

For most use cases, you want your path to include a Guard relay as the first hop—this provides the strongest protection against traffic analysis. The Tor client automatically selects guards based on the consensus, but understanding these flags helps you debug routing issues.

## Troubleshooting Network Issues

When Tor connections fail, the consensus is often the first place to look:

1. **Expired consensus**: If your client hasn't downloaded a fresh consensus (older than 24 hours), it will refuse to use it. Check `tor` logs for "Not sufficiently fresh" errors.

2. **Authority downtime**: If multiple authorities are offline, the consensus may be delayed or have reduced signature quorum.

3. **Relay flag changes**: A relay losing its "Stable" or "Guard" flag mid-session can cause circuit failures. The client should automatically rebuild circuits, but this manifests as intermittent connection drops.

You can monitor the consensus status using the Onionoo API or by checking the Tor metrics portal:

```bash
curl -s "https://onionoo.torproject.org/summary?limit=1" | jq .
```

## Securing Your Bootstrap

For applications that must operate in adversarial environments, consider hardening your bootstrap process:

- **Hardcode authority fingerprints**: Verify the consensus signatures against known-good authority keys rather than trusting any signed document.
- **Use fallback directories**: Configure multiple directory mirrors in case your primary is unreachable.
- **Monitor for consensus manipulation**: Track consensus changes over time to detect unusual modifications.

The Tor network's security model depends on having at least 5 honest authorities. As of 2026, this assumption holds—the nine authorities are operated by diverse, independent organizations. However, for high-value deployments, you should implement the signature verification steps shown above.

## Summary

Directory authorities form the trusted core of Tor's directory system, producing a hourly consensus that every client uses to discover and validate relays. The multi-signed consensus document provides cryptographic guarantees about network state, while the flag system helps clients select reliable paths. For developers, Stem and the Onionoo API provide programmatic access to this infrastructure, enabling sophisticated monitoring and debugging tools.

Understanding how the consensus works is fundamental to working effectively with Tor—whether you're debugging connection issues, building privacy-preserving applications, or simply verifying that your browsing traffic is being routed correctly.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
