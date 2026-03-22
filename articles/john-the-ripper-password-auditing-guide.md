---





































































































































































































































layout: default
title: "How to Use John the Ripper for Password Auditing"
<<<<<<< HEAD
description: "Audit password security with John the Ripper using wordlist attacks, rule-based mangling, incremental mode, and hash extraction from Linux, Windows, and."
=======
description: "Audit password security with John the Ripper using wordlist attacks, rule-based mangling, incremental mode, and hash extraction from Linux, Windows, and
>>>>>>> 900910b9245b9b701f9371a2b27423efb875d01e
date: 2026-03-22
author: "Privacy Tools Guide"
permalink: /john-the-ripper-password-auditing-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---






































































































































































































































{% raw %}
# How to Use John the Ripper for Password Auditing

John the Ripper (JtR) audits the strength of stored password hashes. In a security assessment, you use it to identify accounts using weak or guessable passwords before an attacker does. This guide covers authorized use — auditing your own infrastructure and running penetration tests you have explicit written permission to perform.

**Legal note**: Only run password auditing on systems and hashes you own or have written authorization to test. Unauthorized access to computer systems is a criminal offense in nearly every jurisdiction.

---

## Install John the Ripper (Jumbo)

The "Jumbo" community build supports hundreds more hash types than the stock version:

```bash
# Ubuntu / Debian — build from source for best performance
sudo apt install -y build-essential libssl-dev libgmp-dev \
  libpcap-dev pkg-config libbz2-dev

git clone https://github.com/openwall/john.git /opt/john
cd /opt/john/src
./configure && make -j$(nproc)

# Binary location
ls /opt/john/run/john

# Symlink for convenience
sudo ln -s /opt/john/run/john /usr/local/bin/john

john --version
```

For GPU acceleration (dramatically faster on bcrypt/MD5crypt):

```bash
sudo apt install -y opencl-headers ocl-icd-libopencl1 nvidia-opencl-dev
cd /opt/john/src && ./configure --enable-opencl && make -j$(nproc)
```

---

## Step 1: Extract Password Hashes

### Linux (shadow file)

```bash
# On the target Linux system (requires root)
sudo unshadow /etc/passwd /etc/shadow > /tmp/linux_hashes.txt

# Or just copy the shadow file directly
sudo cp /etc/shadow /tmp/shadow_backup.txt
```

### Windows (NTLM from SAM)

On a Windows system you own, use secretsdump or impacket:

```bash
# From a mounted Windows volume (e.g., forensics scenario)
pip3 install impacket
impacket-secretsdump -system /mnt/windows/Windows/System32/config/SYSTEM \
  -sam /mnt/windows/Windows/System32/config/SAM LOCAL

# Output format: username:RID:LM_hash:NTLM_hash:::
# Save NTLM hashes for John
```

### PostgreSQL

```sql
-- As superuser
SELECT usename, passwd FROM pg_shadow;
-- Hashes start with 'md5' (MD5) or 'SCRAM-SHA-256'
```

### MySQL

```bash
mysql -u root -p -e "SELECT User, authentication_string FROM mysql.user;" \
  > /tmp/mysql_hashes.txt
```

---

## Step 2: Identify Hash Type

```bash
# John auto-detects many types
john --list=formats | grep -i ntlm
john --list=formats | grep -i sha512

# hash-identifier (separate tool)
pip3 install hashid
hashid '$6$rounds=656000$...'
# [+] SHA-512 Crypt

# Common format strings for --format flag:
# sha512crypt    = Linux $6$ (SHA-512)
# sha256crypt    = Linux $5$ (SHA-256)
# bcrypt         = $2a$, $2b$ (Blowfish)
# NT             = Windows NTLM
# md5crypt       = Linux $1$ and FreeBSD MD5
```

---

## Step 3: Wordlist Attack

The fastest attack — tries every word in a dictionary:

```bash
# Download rockyou (most common wordlist)
wget https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt \
  -O /opt/john/run/rockyou.txt

# Run wordlist attack
john --format=sha512crypt --wordlist=/opt/john/run/rockyou.txt /tmp/linux_hashes.txt

# Show cracked passwords
john --show /tmp/linux_hashes.txt

# Show only accounts with cracked passwords
john --show=left /tmp/linux_hashes.txt   # shows remaining uncracked
```

---

## Step 4: Rules-Based Mangling

Rules apply transformations to each wordlist entry — capitalizing, adding numbers, substituting characters. This catches "Password1", "p@ssw0rd", and similar patterns:

```bash
# Use the built-in Jumbo rules (extremely thorough)
john --format=sha512crypt \
  --wordlist=/opt/john/run/rockyou.txt \
  --rules=Jumbo \
  /tmp/linux_hashes.txt

# Use "best64" rules (fast, high yield)
john --format=sha512crypt \
  --wordlist=/opt/john/run/rockyou.txt \
  --rules=best64 \
  /tmp/linux_hashes.txt
```

Write a custom rule in `john.conf`:

```ini
# /opt/john/run/john.conf — add in [List.Rules:Custom] section
[List.Rules:Custom]
# Append 1-4 digits
Az"[0-9]"
Az"[0-9][0-9]"
Az"[0-9][0-9][0-9]"
Az"[0-9][0-9][0-9][0-9]"
# Capitalize first letter, append !
c Az"!"
# l33t substitutions
ss$s
se3e
so0o
```

```bash
john --format=sha512crypt \
  --wordlist=/opt/john/run/rockyou.txt \
  --rules=Custom \
  /tmp/linux_hashes.txt
```

---

## Step 5: Incremental (Brute-Force) Mode

When wordlist attacks fail, try all character combinations within a keyspace:

```bash
# All lowercase alpha up to 6 chars (Alpha6 built-in)
john --format=sha512crypt --incremental=Alpha /tmp/linux_hashes.txt

# All digits (for PIN-style passwords)
john --format=sha512crypt --incremental=Digits /tmp/linux_hashes.txt

# Custom character set — printable ASCII, max 8 chars
# Define in john.conf:
# [Incremental:Printable8]
# File = $JOHN/printable.chr
# MaxLen = 8
# MinLen = 1

john --format=NT --incremental=Printable8 /tmp/ntlm_hashes.txt
```

---

## Step 6: Run Multiple Sessions in Parallel

```bash
# Split hash file and run two instances on different CPUs
split -n l/2 /tmp/linux_hashes.txt /tmp/split_

john --format=sha512crypt \
  --session=session1 \
  --wordlist=/opt/john/run/rockyou.txt \
  --rules=Jumbo \
  /tmp/split_aa &

john --format=sha512crypt \
  --session=session2 \
  --wordlist=/opt/john/run/rockyou.txt \
  --rules=Jumbo \
  /tmp/split_ab &

# Check status of a running session
john --status=session1
```

---

## Step 7: Crack NTLM Hashes (Windows)

NTLM hashes are unsalted and fast to crack — thousands per second on a GPU:

```bash
# Format: user:rid:lm:ntlm:::
echo "administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::" \
  > /tmp/ntlm.txt

# Wordlist + rules
john --format=NT \
  --wordlist=/opt/john/run/rockyou.txt \
  --rules=Jumbo \
  /tmp/ntlm.txt

# Show results
john --format=NT --show /tmp/ntlm.txt
```

---

## Step 8: Read Results and Write an Audit Report

```bash
# All cracked credentials
john --show /tmp/linux_hashes.txt

# Summary
echo "Total hashes: $(wc -l < /tmp/linux_hashes.txt)"
echo "Cracked: $(john --show /tmp/linux_hashes.txt | tail -1)"

# Identify accounts using duplicate passwords
john --show /tmp/linux_hashes.txt | awk -F: '{print $2}' | sort | uniq -d
```

Document findings with:
- Account name
- Password complexity (weak/medium — do not log the actual password in reports)
- Attack vector used (wordlist, rules, brute-force)
- Recommendation: enforce minimum 12-char policy, MFA, credential rotation

---

## Defending Against Cracking

The best defense against offline hash cracking:

1. Use **bcrypt** (cost factor 12+), **Argon2id**, or **scrypt** — not MD5/SHA-1/NTLM
2. Enforce **minimum 16 characters** for admin accounts
3. Enable **MFA** so cracked passwords alone aren't sufficient
4. Use **hibp-go** or similar to reject passwords in known breach datasets at sign-up

```python
# Check against HaveIBeenPwned k-anonymity API
import hashlib, httpx

def is_pwned_password(password: str) -> int:
    digest = hashlib.sha1(password.encode()).hexdigest().upper()
    prefix, suffix = digest[:5], digest[5:]
    resp = httpx.get(f"https://api.pwnedpasswords.com/range/{prefix}")
    for line in resp.text.splitlines():
        h, count = line.split(":")
        if h == suffix:
            return int(count)   # number of times seen in breaches
    return 0
```

---

## Related Reading

- [How to Use Volatility for Memory Forensics](/privacy-tools-guide/volatility-memory-forensics-guide/)
- [How to Use Metasploit for Authorized Pentesting](/privacy-tools-guide/metasploit-authorized-pentesting-guide/)
- [How to Use Nessus for Vulnerability Scanning](/privacy-tools-guide/nessus-vulnerability-scanning-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
