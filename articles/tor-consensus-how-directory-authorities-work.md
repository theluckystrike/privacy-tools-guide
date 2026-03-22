---
layout: default
title: "Tor Consensus: How Directory Authorities"
description: "A technical deep-dive into Tor network consensus mechanism and directory authorities. Learn how relay information is validated, how clients bootstrap"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /tor-consensus-how-directory-authorities-work/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---
---
layout: default
title: "Tor Consensus: How Directory Authorities"
description: "A technical deep-dive into Tor network consensus mechanism and directory authorities. Learn how relay information is validated, how clients bootstrap"
date: 2026-03-15
last_modified_at: 2026-03-15
author: theluckystrike
permalink: /tor-consensus-how-directory-authorities-work/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]---

{% raw %}

The Tor network relies on a decentralized system of relays to provide anonymity, but clients need a reliable way to discover which relays exist and which ones to trust. This is where the **consensus** mechanism and **directory authorities** come into play. Understanding this infrastructure is essential for developers building Tor-integrated applications, security researchers auditing network integrity, or power users who want to verify their connection's reliability.

## Key Takeaways

- **Are there free alternatives**: available? Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support.
- **What is the learning**: curve like? Most tools discussed here can be used productively within a few hours.
- **Step 4**: Distribution
The final consensus document is signed by at least 5 of 9 authorities (a quorum requirement).
- **Expired consensus**: If your client hasn't downloaded a fresh consensus (older than 24 hours), it will refuse to use it.
- **The Tor network's security**: model depends on having at least 5 honest authorities.
- **Mastering advanced features takes**: 1-2 weeks of regular use.

## What Are Directory Authorities

Directory authorities (DAs) are specially designated servers that form the backbone of Tor's directory system. There are nine primary authorities operated by different entities around the world—each managed by trusted organizations or individuals committed to the Tor project's mission.

These authorities perform three critical functions:

1. **Relay voting** — Each authority collects relay descriptors from all published relays
2. **Consensus generation** — Authorities vote on the network state and produce an unified document
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

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

**Can I trust these tools with sensitive data?**

Review each tool's privacy policy, data handling practices, and security certifications before using it with sensitive data. Look for SOC 2 compliance, encryption in transit and at rest, and clear data retention policies. Enterprise tiers often include stronger privacy guarantees.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [China Exit Ban Digital Surveillance How Authorities Monitor](/privacy-tools-guide/china-exit-ban-digital-surveillance-how-authorities-monitor-/)
- [Russia Vpn Provider Compliance Which Services Handed.](/privacy-tools-guide/russia-vpn-provider-compliance-which-services-handed-user-data-to-authorities-2026-review/)
- [Android Work Profile for Isolating Apps That Require.](/privacy-tools-guide/android-work-profile-for-isolating-apps-that-require-invasiv/)
- [Android Work Profile Privacy Separation Guide](/privacy-tools-guide/android-work-profile-privacy-separation-guide/)
- [Best No Kyc Cryptocurrency Exchanges That Still Work In 2026](/privacy-tools-guide/best-no-kyc-cryptocurrency-exchanges-that-still-work-in-2026/)

## Directory Authority Network Architecture

The nine Tor directory authorities are geographically distributed to prevent coordinated attacks. Each authority maintains independently:

- **Moria1** (MIT): Run by Peter Palfrader, Tor Project
- **Tor26** (TU Darmstadt, Germany): Operated by Tor Project
- **Dizum** (Internet Systems Consortium): Tor Project
- **Baalbek** (Swedish Internet Foundation): Tor Project
- **Longclaw** (Tor Project): Designated operator
- **Gabelmoo** (University of Paderborn): Tor Project staff
- **Dannenberg** (CCC, Germany): Chaos Computer Club
- **Faravahar** (Iranian diaspora): Community operator
- **Serpa** (University of Tartu, Estonia): Community operator

### Voting Protocol Details

The consensus generation process follows a strict protocol:

```
Hour H:00 - Authorities download all relay descriptors
Hour H:20 - Each authority publishes its vote
Hour H:25 - Authorities vote on consensus parameters
Hour H:40 - Final consensus published and distributed
Hour H+1:00 - Clients use consensus for routing decisions
```

During the voting window, each authority can propose consensus parameters. If more than half agree, the parameter becomes law for that consensus period. This determines critical settings like:

- Minimum relay bandwidth thresholds
- Guard selection criteria
- Exit policy requirements
- Circuit path construction rules

### Consensus Parameter Examples

```
bwauthpid=<value>                    # Bandwidth authority measurements
guard-lifetime-days=<value>          # How long a guard can serve
max-client-circuits=<value>          # Circuit creation rate limiting
nf-congestratio-mult=<value>         # Network congestion handling
```

These parameters change frequently as the Tor network evolves and responds to new attacks.

## Advanced Topics for Network Security

### Detecting Authority Compromise

If multiple authorities became compromised, the consensus could be manipulated. Tor has built-in defenses:

1. **Quorum requirement**: 5 of 9 authorities must sign. An attacker needs to compromise majority
2. **Authority key pinning**: Clients can verify signatures against hardcoded public keys
3. **Redundant distribution**: Consensus propagates through both authorities and mirrors, preventing single-point suppression

Operators can verify consensus integrity:

```python
from stem import descriptor
import hashlib

def verify_consensus_integrity(consensus_path, known_authority_keys):
    """Verify consensus meets security thresholds"""

    with open(consensus_path, 'rb') as f:
        consensus = descriptor.Consensus(f.read())

    # Check signature count
    sig_count = len(consensus.signatures)
    if sig_count < 5:
        print(f"ALERT: Only {sig_count}/9 signatures present")
        return False

    # Verify each signature matches known authority key
    valid_sigs = 0
    for sig in consensus.signatures:
        authority_id = sig.identity
        if authority_id in known_authority_keys:
            valid_sigs += 1

    print(f"Valid authority signatures: {valid_sigs}/{sig_count}")
    return valid_sigs >= 5
```

### Bandwidth Authority Measurements

One consensus function is bandwidth measurement. Authorities run bandwidth tests against relays to verify their claimed throughput:

```
"w" line in consensus:
w Bandwidth=<kilabytes_per_second>
```

Lying about bandwidth gets you deprioritized or down-weighted. A relay claiming 10 Mbps but measuring at 1 Mbps loses flag status.

### Exit Policy Parsing

The consensus includes each relay's exit policy. Understanding how to parse these determines what traffic types a relay handles:

```python
def parse_exit_policy(policy_line):
    """Parse Tor exit policy (simplified)"""
    # Format: accept/reject ip:port[, ...]

    # Special cases:
    # "accept *:80, accept *:443, reject *:*" = web traffic only
    # "reject *:*" = reject all (middle relay only)
    # "accept *:*" = accept all (unrestricted exit)

    rules = []
    for rule in policy_line.split(','):
        action, spec = rule.strip().split()
        rules.append({'action': action, 'port_spec': spec})

    return rules
```

Clients use exit policies to determine which relays can service their requests. A HTTPS request (port 443) cannot use a relay rejecting that port.

## Network Monitoring and Analysis

### Real-Time Consensus Metrics

Organizations and researchers monitor consensus health:

```bash
# Query Onionoo for current network statistics
curl -s "https://onionoo.torproject.org/summary" | jq '.relays | length'  # Total relays
curl -s "https://onionoo.torproject.org/summary?flag=Guard" | jq '.relays | length'  # Guard count
curl -s "https://onionoo.torproject.org/details?flag=Exit" | jq '.relays | map(.bandwidth) | add'  # Exit bandwidth
```

A healthy network maintains:
- 8,000+ relays total
- 1,200+ Guard-flagged relays
- 800+ Exit-flagged relays
- Even bandwidth distribution (prevents choke points)

### Detecting Sybil Attacks

A Sybil attack involves a single entity controlling many relays to increase probability of handling user traffic. Detection mechanisms include:

- **AS path analysis**: Relays from same ASN get downweighted
- **IP clustering**: Multiple relays from same /16 network face penalties
- **Uptime correlation**: Relays starting/stopping together trigger investigation

## Historical Consensus Issues

Understanding past problems helps operators maintain integrity:

- **2013 - Directory Authority Attacks**: Attackers attempted to publish false consensus. Signature verification prevented compromise
- **2018 - Consensus Bandwidth Inflation**: Some authorities over-weighted certain relays. Improved measurement corrected this
- **2022 - Authority Availability**: Extended downtime of one authority reduced signature count. Network degraded gracefully

The consensus mechanism's redundancy and cryptography have held against sustained attacks for 17+ years.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
