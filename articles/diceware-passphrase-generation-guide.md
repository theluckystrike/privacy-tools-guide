---
layout: default
title: "Diceware Passphrase Generation Guide"
description: "How to generate cryptographically strong passphrases using Diceware and EFF wordlists, with entropy calculations, offline dice rolling, and digital tools"
date: 2026-03-21
author: theluckystrike
permalink: /diceware-passphrase-generation-guide/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

A passphrase like `correct-horse-battery-staple` is both memorable and cryptographically strong. A password like `P@ssw0rd!23` is neither. Diceware is the method for generating random passphrases with precisely calculable entropy — you roll physical dice and look up words in a numbered wordlist. The result is provably random, requires no trust in any software, and produces passphrases strong enough for full disk encryption master keys.

## What Makes Diceware Secure

Diceware's strength comes from genuine randomness — physical dice rolls — combined with a known wordlist. Because the wordlist and the number of dice rolls are public, you can calculate exactly how hard the passphrase is to crack.

Each roll of five dice produces a number from 11111 to 66666, mapping to one of 7,776 possible words (6^5 = 7,776). Each word adds log₂(7,776) ≈ 12.9 bits of entropy.

| Words | Entropy | Time to crack at 1 trillion guesses/second |
|-------|---------|-------------------------------------------|
| 4 | ~51 bits | ~27 days |
| 5 | ~64 bits | ~550 years |
| 6 | ~77 bits | ~4 million years |
| 7 | ~90 bits | ~37 billion years |
| 8 | ~103 bits | ~300 trillion years |

For most uses (disk encryption, password manager master password, PGP key passphrase), 6 words provides ample security. For high-value keys that will persist for decades, use 7 or 8 words.

## What You Need

- Five standard dice (6-sided)
- The EFF Long Wordlist (preferred) or original Diceware wordlist
- Pen and paper

Download the EFF wordlist:

```bash
curl -O https://www.eff.org/files/2016/07/18/eff_large_wordlist.txt
```

The EFF wordlist was created specifically for Diceware and contains common, readable English words that are easier to memorize than the original list.

## Generating a Passphrase Offline

Roll all five dice simultaneously (or sequentially — the order matters). Write down the five numbers in the order they land, left to right, to get a five-digit number.

Example rolls for a 6-word passphrase:

```
Roll 1: 3 5 1 4 2 → 35142 → "laden"
Roll 2: 1 6 5 3 2 → 16532 → "corral"
Roll 3: 4 2 1 6 5 → 42165 → "mulch"
Roll 4: 5 5 1 2 4 → 55124 → "scone"
Roll 5: 3 1 4 5 6 → 31456 → "gusto"
Roll 6: 6 2 3 1 1 → 62311 → "tweed"
```

Resulting passphrase: `laden corral mulch scone gusto tweed`

To look up a word: open `eff_large_wordlist.txt` and find the line starting with your five-digit number.

```bash
grep "^35142" eff_large_wordlist.txt
# 35142	laden
```

## Generating Passphrases Digitally

If physical dice aren't available, use a trusted tool that sources randomness from the OS's cryptographically secure RNG:

### Python (uses os.urandom internally via secrets module)

```python
#!/usr/bin/env python3
import secrets

def generate_passphrase(wordlist_path, num_words=6):
    # Load wordlist
    with open(wordlist_path) as f:
        words = [line.split('\t')[1].strip() for line in f if '\t' in line]

    if len(words) < 7776:
        raise ValueError(f"Expected 7776 words, got {len(words)}")

    # Generate passphrase
    passphrase = [secrets.choice(words) for _ in range(num_words)]
    return ' '.join(passphrase)

# Usage
wordlist = 'eff_large_wordlist.txt'
print(generate_passphrase(wordlist, num_words=6))
```

```bash
python3 generate_passphrase.py
# Example output: violet margin dapper cactus lemon bronze
```

### Using `diceware` command-line tool

```bash
pip3 install diceware

# Generate 6-word passphrase
diceware -n 6

# Use EFF wordlist
diceware -n 6 --wordlist en_eff

# With custom separator
diceware -n 6 --delimiter '-'
```

### Using /dev/urandom directly (for shell scripting)

```bash
#!/bin/bash
WORDLIST="eff_large_wordlist.txt"
NUM_WORDS=6

for i in $(seq 1 $NUM_WORDS); do
    # Generate random 5-digit dice number (each digit 1-6)
    dice=""
    for d in $(seq 1 5); do
        die=$(( ($(od -An -N1 -tu1 /dev/urandom | tr -d ' ') % 6) + 1 ))
        dice="${dice}${die}"
    done
    grep "^${dice}" "$WORDLIST" | awk '{print $2}'
done | tr '\n' ' '
echo
```

## Entropy Comparison: Passphrases vs Passwords

Common password policies produce weak passwords with poor entropy:

| Format | Example | Entropy |
|--------|---------|---------|
| 8-char password, mixed case + symbols | `P@ss0w!d` | ~40-50 bits |
| 12-char random password | `aB3#mKp2!xQz` | ~72 bits |
| 4-word Diceware | `levy comet radar wing` | ~51 bits |
| 6-word Diceware | `levy comet radar wing pulp damp` | ~77 bits |
| 8-word Diceware | `levy comet radar wing pulp damp sock fern` | ~103 bits |

A 12-character random password is strong but nearly impossible to memorize. A 6-word Diceware passphrase has similar entropy and is much more memorable.

## When to Use Passphrases vs Random Passwords

**Use Diceware passphrases for:**
- Password manager master password (must be memorizable, very high stakes)
- Full disk encryption (LUKS, BitLocker, VeraCrypt)
- PGP/GPG private key passphrase
- SSH private key passphrase
- Any credential you must type regularly from memory

**Use randomly generated passwords for:**
- Individual website accounts stored in a password manager (memorability doesn't matter)
- API keys and tokens (never typed by humans)
- Any credential where a password manager handles entry

## Memorizing a Diceware Passphrase

The key technique is **spaced repetition** combined with a memory device:

1. Generate your 6-word passphrase
2. Create a vivid mental image connecting all 6 words in sequence (the weirder, the more memorable)
3. Type it 5 times slowly right after generating it
4. Type it again 30 minutes later, then 4 hours later, then the next morning
5. After a week of daily use it will be automatic

Example: `laden corral mulch scone gusto tweed`

Mental image: A farmer **laden** with bags walks into a **corral**, steps in **mulch**, tries to eat a **scone**, but chokes from **gusto**, and is wearing **tweed**. Absurd images stick better than coherent ones.

## When Physical Dice Are Compromised

A common question: if dice can be loaded, is this method secure?

Standard dice rolls have about ±0.2% bias per face. For Diceware, this means the least-likely word appears with probability ~0.998^5 ≈ 0.99 of normal and the most likely with probability ~1.002^5 ≈ 1.01 of normal. This bias is negligibly small — it reduces entropy by a fraction of a bit across all 6 words.

For situations where you want certainty, use the Python `secrets` module implementation above — it uses the OS's CSPRNG which is audited and tested against bias.

## Threat Model: When Diceware Isn't Enough

For certain threat models, even 8-word Diceware passphrases may be insufficient. Consider using longer passphrases for scenarios involving:

- **Cryptocurrency cold wallets**: Keys that control significant value should use 10+ words
- **Full disk encryption protecting highly sensitive data**: 8-10 words minimum
- **Master keys for entire digital identities**: 12 words provides headroom against future computing advances

To estimate future security, calculate bits of entropy needed:

| Threat Model | Time Horizon | Entropy Required | Diceware Words |
|--------------|--------------|------------------|-----------------|
| Standard account protection | 5 years | 64 bits | 5-6 words |
| Financial assets | 10 years | 80 bits | 7 words |
| Secrets lasting decades | 20+ years | 100+ bits | 8-9 words |
| Quantum-resistant equivalent | 30+ years | 256 bits | 20+ words |

The last row highlights a critical limitation: if cryptographically broken passphrase hashes are captured today and quantum computers emerge in 20 years, no amount of entropy protects you. Use additional protections like time-locked encryption for long-term secrets.

## Advanced: Diceware with Passphrase Stretching

Raw Diceware provides excellent entropy but no computational cost to attackers. Password stretching functions like PBKDF2 or Argon2 make brute-force attacks exponentially harder:

```python
#!/usr/bin/env python3
import hashlib
import hmac
import os

def stretch_passphrase(passphrase, iterations=200000):
    """
    Stretch a Diceware passphrase using PBKDF2.
    Increases computational cost for attackers.
    """
    salt = os.urandom(32)

    stretched = hashlib.pbkdf2_hmac(
        'sha256',
        passphrase.encode(),
        salt,
        iterations
    )

    return stretched.hex(), salt.hex()

# Example
passphrase = "laden corral mulch scone gusto tweed"
stretched, salt = stretch_passphrase(passphrase)
print(f"Stretched: {stretched[:32]}...")
print(f"Salt: {salt}")

# To recover: repeat with same salt and iteration count
```

Use stretched passphrases in applications where you control the stretching function. For applications like LUKS disk encryption, use the application's built-in key stretching (which it does automatically).

## Verifying Randomness Quality

Before committing a generated passphrase to long-term use, verify the randomness source:

```bash
#!/bin/bash
# Test /dev/urandom for quality randomness

# Extract 1MB of random data
dd if=/dev/urandom of=/tmp/random.bin bs=1M count=1 2>/dev/null

# Run entropy analysis (requires ent tool)
ent /tmp/random.bin

# Expected output:
# Entropy = 7.999972 bits per byte (close to 8.0 is good)
# Chi-square = 234.5 (closer to 256 is better)
```

Entropy close to 8 bits per byte and chi-square values near 256 indicate high-quality randomness. Values significantly different may suggest problems with your random source.

## Diceware for Multiple Languages

The EFF wordlist exists in English, but multiple language implementations are available:

| Language | List Size | Words | Source |
|----------|-----------|-------|--------|
| English (EFF) | 7776 | Common, readable | eff.org |
| German | 7776 | Diceware original | Benutzerhandbuch |
| Italian | 7776 | Modern, accessible | Italian crypto community |
| Spanish | 7776 | Natural phrases | Available on GitHub |
| French | 7776 | Phonetically distinct | French EFF equivalent |

For international teams, coordinating on a single language (typically English) prevents confusion. Translated wordlists offer benefits for teams in non-English speaking regions who struggle to memorize English word sequences.

## Related Reading

- [Two-Factor Authentication Setup 2026](/two-factor-authentication-setup-2026/)
- [YubiKey Setup for Multiple Services Guide](/yubikey-setup-multiple-services-guide/)
- [LUKS Full Disk Encryption on Linux](/luks-full-disk-encryption-linux-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
