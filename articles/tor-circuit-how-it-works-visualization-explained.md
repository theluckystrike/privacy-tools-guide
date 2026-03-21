---
layout: default
title: "Tor Circuit: How It Works and Visualization Explained"
description: "A technical deep dive into Tor circuit mechanics for developers and power users. Understand the onion routing protocol, circuit construction, and how"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /tor-circuit-how-it-works-visualization-explained/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Tor circuits route your traffic through exactly three relays—the guard node (knows your IP but not destination), middle node (knows neither origin nor destination), and exit node (knows destination but not origin)—such that no single relay sees your complete path. This three-hop design encrypts traffic at each layer and balances anonymity with performance, making it computationally infeasible for passive adversaries to trace your connection from origin to destination even if they control some network infrastructure.

## The Three-Hop Tor Circuit

Every Tor connection passes through exactly three relays: the entry guard (or guard node), the middle node, and the exit node. This three-hop design provides a balance between anonymity and performance.

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐      ┌─────────────────┐
│   Client    │ ───► │   Guard     │ ───► │   Middle    │ ───► │     Exit        │
│  (Your IP)  │      │   Node      │      │   Node      │      │     Node        │
└─────────────┘      └─────────────┘      └─────────────┘      └─────────────────┘
       │                   │                   │                   │
       │ ◄─ Encrypted ─►   │ ◄─ Encrypted ─►   │ ◄─ Encrypted ─►   │
       │                   │                   │                   │
       │            Knows only             Knows only            Knows origin
       │            client IP               guard IP             (via TLS metadata)
```

### Why Three Relays?

Using three hops rather than more provides several advantages:

- **Performance**: Each hop adds latency. Three hops keep latency manageable (typically 100-500ms additional delay).
- **Anonymity**: No single relay knows both where your traffic originated and where it's going.
- **Attack surface reduction**: An adversary controlling fewer nodes has lower probability of compromising the entire circuit.

## Circuit Construction Process

The Tor protocol uses a sophisticated key exchange mechanism to build circuits. Here's how it works:

### 1. Establishing the Circuit

When you connect to Tor, your client performs a Diffie-Hellman key exchange with each relay in sequence:

```python
# Simplified circuit extension (pseudo-code)
def extend_circuit(circuit, relay_info):
    # Create handshake with new relay
    public_key = generate_keypair()
    
    # Diffie-Hellman with relay
    shared_secret = diffie_hellman(
        public_key,
        relay_info.identity_key
    )
    
    # Derive session keys
    keys = derive_keys(shared_secret)
    
    # Add hop to circuit
    circuit.add_hop(relay_info, keys)
    
    return circuit
```

### 2. Onion Encryption Layers

Each relay only decrypts one layer of encryption, revealing only where to forward the traffic next. This creates the "onion" effect:

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 3 (Exit Node) ◄── Decrypts this layer               │
│  ─────────────────────────────────────────────────────────  │
│  Layer 2 (Middle Node) ◄── Decrypts this layer             │
│  ─────────────────────────────────────────────────────────  │
│  Layer 1 (Guard Node) ◄── Decrypts this layer              │
│  ─────────────────────────────────────────────────────────  │
│  Layer 0 (Your Client) ◄── Original encrypted payload     │
└─────────────────────────────────────────────────────────────┘
```

### 3. Cell Structure

Tor uses fixed-size cells (512 bytes) for all communications. Each cell contains:

- Circuit ID (identifies the circuit)
- Relay command (EXTEND, DATA, DESTROY, etc.)
- Payload (encrypted data)

```c
// Tor cell structure (simplified)
struct tor_cell {
    uint16_t circuit_id;    // Circuit identifier
    uint8_t  command;       // RELAY_BEGIN, RELAY_DATA, etc.
    uint8_t  payload[509];  // Encrypted payload
};
```

## Visualizing Circuit Paths

You can visualize your current Tor circuit using the Tor Browser or by querying the Tor control port.

### Using Tor Browser

1. Click the shield icon in the Tor Browser toolbar
2. View the "Circuit" section to see your current path
3. Each relay shows its nickname, IP (partially obscured), and country

### Programmatic Access

Connect to the Tor control port to programmatically inspect circuits:

```bash
# Enable control port in torrc
# ControlPort 9051
# CookieAuthentication 1

# Connect via socat or netcat
echo -e "AUTHENTICATE\r\nGETINFO circuit-status\r\nQUIT" | nc localhost 9051
```

```python
# Using Stem library (Python)
from stem import Controller

with Controller.from_port(port=9051) as controller:
    controller.authenticate()
    
    # Get all circuits
    for circuit in controller.get_circuits():
        print(f"Circuit {circuit.id}:")
        for i, hop in enumerate(circuit.path):
            print(f"  Hop {i+1}: {hop[0]} ({hop[1]})")
```

This outputs something like:

```
Circuit 12:
  Hop 1: Unnamed (86.123.45.67) [Germany]
  Hop 2: DCHubRelays03 (193.56.78.90) [Netherlands]  
  Hop 3: tor4you (45.67.89.123) [France]
```

## Circuit Lifetime and Renewal

Tor circuits don't last forever. The default circuit lifetime is 10 minutes, after which Tor builds a new circuit. This limits the amount of traffic correlation an attacker can perform.

### Configuration Options

You can adjust circuit behavior in your `torrc`:

```bash
# Build new circuit every N seconds (default: 600)
MaxCircuitDirtime 600

# Number of concurrent circuits (default: 1)
MaxClientCircuitsPending 1

# Use only entry guards that have been stable for N days
NumEntryGuards 3
```

## Understanding Guard Nodes

Guard nodes (also called entry guards) are a critical security feature. Instead of choosing random entry points each time, Tor clients select a small set of stable relays as permanent guards.

This prevents certain attacks:

- **First-hop correlation**: Without guards, an attacker running many relays could become your entry node frequently
- **Traffic analysis**: Longer guard tenure means less opportunity for fingerprinting

### Guard Selection Algorithm

```python
# Simplified guard selection logic
def select_guards(relays, config):
    # Filter stable, fast relays
    candidates = [r for r in relays 
                  if r.is_stable and r.is_fast 
                  and r.bandwidth > config.min_bandwidth]
    
    # Select weighted by bandwidth
    guards = weighted_selection(candidates, config.num_guards)
    
    return guards
```

## Exit Node Considerations

The exit node is where Tor traffic meets the regular internet. This position carries unique risks:

- **TLS metadata exposure**: The exit node sees the destination domain (via SNI in TLS handshake)
- **Content inspection**: Unencrypted traffic can be observed
- **Exit policy**: Each relay has its own exit policy determining what connections it will relay

### Common Exit Node Ports

Most Tor relays allow these common ports:

```bash
# Typical exit policy (most permissive)
accept *:80, *:443    # HTTP, HTTPS
accept *:22           # SSH
accept *:853          # DNS over TLS
reject *:*            # Reject everything else
```

## Building Applications with Tor

For developers integrating Tor, the Stem library provides excellent Python bindings:

```python
from stem import Signal
from stem.control import Controller

# Start a new Tor circuit for each request
def tor_request(url, session):
    with Controller.from_port(port=9051) as controller:
        controller.authenticate()
        
        # Signal Tor to create new identity
        controller.signal(Signal.NEWNYM)
        
        # Configure session to use Tor
        session.proxies = {
            'http': 'socks5h://127.0.0.1:9050',
            'https': 'socks5h://127.0.0.1:9050'
        }
        
        return session.get(url)
```

## Security Considerations

Understanding Tor circuit limitations helps you use it effectively:

- **Timing attacks**: Correlating traffic timing at multiple points can de-anonymize users
- **Exit node logging**: Some exit nodes log traffic; assume all traffic may be observed
- **Protocol leakage**: Applications must be configured to avoid leaking identifying information outside the Tor circuit
- **Circuit fingerprinting**: Traffic patterns through a circuit can be fingerprinted



## Related Reading

- [How Browser Fingerprinting Works Explained](/privacy-tools-guide/how-browser-fingerprinting-works-explained/)
- [Arti Tor Rust Implementation Explained 2026](/privacy-tools-guide/arti-tor-rust-implementation-explained-2026/)
- [Nym Mixnet vs Tor Comparison Explained: A Technical Guide](/privacy-tools-guide/nym-mixnet-vs-tor-comparison-explained/)
- [Tor Browser Threat Model Explained for Developers](/privacy-tools-guide/tor-browser-threat-model-explained-developers/)
- [Tor Network Censorship Resistance Explained](/privacy-tools-guide/tor-network-censorship-resistance-explained/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
