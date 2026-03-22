---
layout: default
title: "How to Set Up a Dead Man's Switch for Data"
description: "Build a Python-based dead man's switch that automatically releases encrypted data or sends alerts if you fail to check in, with configurable escalation logic"
date: 2026-03-22
author: theluckystrike
permalink: /dead-mans-switch-python-script-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Set Up a Dead Man's Switch for Data

A dead man's switch (DMS) releases information or sends alerts automatically if you fail to check in within a set period. Uses include: releasing encrypted credentials to trusted people if you're incapacitated, notifying contacts if you go silent, or deleting sensitive data if a device isn't checked within a timeframe.

This guide builds a Python-based DMS with configurable check-in periods, escalating alerts, and encrypted payload release.
---

## Table of Contents

- [Design Principles](#design-principles)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Troubleshooting](#troubleshooting)
- [Related Reading](#related-reading)

## Design Principles

A good DMS has:
1. **Regular check-in requirement**: You must actively confirm you're alive at intervals
2. **Escalating response**: First sends a reminder, then alerts, then releases/acts
3. **Tamper resistance**: Stored on a system you don't fully control (or on multiple systems)
4. **Encrypted payload**: The released data is encrypted until needed
5. **Clear recovery**: Easy for intended recipients to use the released data

---

## Architecture

```
[You] ← regularly send check-in token → [DMS Server/Script]
                                               |
                              (if no check-in within window)
                                               |
                              [1] Email reminder to you
                              [2] Alert to trusted contact
                              [3] Release encrypted payload
                              [4] Delete local sensitive data
```

---

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Core DMS Script

```python
#!/usr/bin/env python3
"""
Dead Man's Switch — checks if you've checked in recently.
Run via cron every hour on a server or trusted machine.
"""
import os
import json
import time
import hashlib
import smtplib
import logging
from datetime import datetime, timedelta
from email.message import EmailMessage
from pathlib import Path

# Configuration
CONFIG = {
    "check_in_file": "/var/dms/last_checkin.json",
    "state_file": "/var/dms/state.json",
    "payload_file": "/var/dms/payload.enc",       # Encrypted payload to release
    "log_file": "/var/log/dms.log",

    # Time windows (hours)
    "reminder_after_hours": 48,     # Send you a reminder after 2 days no check-in
    "alert_after_hours": 96,        # Alert trusted contacts after 4 days
    "release_after_hours": 168,     # Release payload after 7 days

    # Your email (where reminder goes)
    "owner_email": "you@protonmail.com",

    # Trusted contacts (receive alert and payload)
    "trusted_contacts": [
        {"name": "Alice", "email": "alice@example.com"},
        {"name": "Bob", "email": "bob@example.com"},
    ],

    # SMTP settings (use an email relay or local MTA)
    "smtp_host": "localhost",
    "smtp_port": 25,
    "smtp_from": "dms@yourserver.com",
    "smtp_user": "",    # leave empty for unauthenticated local relay
    "smtp_pass": "",
}

logging.basicConfig(
    filename=CONFIG["log_file"],
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s"
)

def load_last_checkin():
    """Return datetime of last check-in, or epoch if no record."""
    p = Path(CONFIG["check_in_file"])
    if not p.exists():
        return datetime.fromtimestamp(0)
    data = json.loads(p.read_text())
    return datetime.fromtimestamp(data.get("timestamp", 0))

def load_state():
    p = Path(CONFIG["state_file"])
    if not p.exists():
        return {}
    return json.loads(p.read_text())

def save_state(state: dict):
    Path(CONFIG["state_file"]).write_text(json.dumps(state, indent=2))

def send_email(to: str, subject: str, body: str, attachment_path: str = None):
    msg = EmailMessage()
    msg["From"] = CONFIG["smtp_from"]
    msg["To"] = to
    msg["Subject"] = subject
    msg.set_content(body)

    if attachment_path and Path(attachment_path).exists():
        with open(attachment_path, "rb") as f:
            msg.add_attachment(f.read(), maintype="application", subtype="octet-stream",
                             filename=Path(attachment_path).name)

    try:
        with smtplib.SMTP(CONFIG["smtp_host"], CONFIG["smtp_port"]) as s:
            if CONFIG["smtp_user"]:
                s.login(CONFIG["smtp_user"], CONFIG["smtp_pass"])
            s.send_message(msg)
        logging.info(f"Email sent to {to}: {subject}")
    except Exception as e:
        logging.error(f"Failed to send email to {to}: {e}")

def run_check():
    last_checkin = load_last_checkin()
    now = datetime.now()
    hours_since = (now - last_checkin).total_seconds() / 3600
    state = load_state()

    logging.info(f"Check-in status: last={last_checkin}, hours_since={hours_since:.1f}")

    # Stage 1: Send owner reminder
    if hours_since >= CONFIG["reminder_after_hours"] and not state.get("reminder_sent"):
        send_email(
            CONFIG["owner_email"],
            "DMS: Check-in required",
            f"Your dead man's switch hasn't received a check-in in {hours_since:.0f} hours.\n"
            f"Check in now: curl -X POST https://yourserver.com/dms/checkin?token=YOUR_TOKEN\n"
            f"Next escalation in {CONFIG['alert_after_hours'] - hours_since:.0f} hours."
        )
        state["reminder_sent"] = True
        state["reminder_time"] = now.isoformat()
        save_state(state)

    # Stage 2: Alert trusted contacts
    if hours_since >= CONFIG["alert_after_hours"] and not state.get("alert_sent"):
        for contact in CONFIG["trusted_contacts"]:
            send_email(
                contact["email"],
                f"DMS Alert: {CONFIG['owner_email']} has not checked in",
                f"This is an automated alert from {CONFIG['owner_email']}'s dead man's switch.\n\n"
                f"There has been no check-in for {hours_since:.0f} hours.\n"
                f"Please attempt to contact them directly.\n\n"
                f"If no check-in occurs, encrypted data will be released in "
                f"{CONFIG['release_after_hours'] - hours_since:.0f} more hours."
            )
        state["alert_sent"] = True
        state["alert_time"] = now.isoformat()
        save_state(state)

    # Stage 3: Release encrypted payload
    if hours_since >= CONFIG["release_after_hours"] and not state.get("payload_released"):
        for contact in CONFIG["trusted_contacts"]:
            send_email(
                contact["email"],
                f"DMS: Encrypted payload from {CONFIG['owner_email']}",
                f"No check-in received for {hours_since:.0f} hours. Releasing encrypted payload.\n\n"
                f"The attached file is encrypted. Decryption instructions were given to you separately.\n"
                f"Use: gpg --decrypt payload.enc",
                attachment_path=CONFIG["payload_file"]
            )
        state["payload_released"] = True
        state["release_time"] = now.isoformat()
        save_state(state)
        logging.warning(f"PAYLOAD RELEASED after {hours_since:.0f}h without check-in")

if __name__ == "__main__":
    run_check()
```

---

### Step 2: Check-In Server (Simple HTTP Endpoint)

```python
#!/usr/bin/env python3
"""Simple Flask endpoint to accept check-in tokens."""
from flask import Flask, request, jsonify
import json, hashlib, time
from pathlib import Path

app = Flask(__name__)

# Set your check-in token (share only with yourself)
# Generate: python3 -c "import secrets; print(secrets.token_urlsafe(32))"
VALID_TOKEN = "YOUR_SECRET_CHECK_IN_TOKEN"
CHECK_IN_FILE = "/var/dms/last_checkin.json"

@app.route("/dms/checkin", methods=["POST", "GET"])
def checkin():
    token = request.args.get("token") or request.form.get("token")

    if not token:
        return jsonify({"error": "Missing token"}), 400

    # Constant-time comparison to prevent timing attacks
    if not hmac.compare_digest(token, VALID_TOKEN):
        return jsonify({"error": "Invalid token"}), 403

    # Record check-in
    data = {"timestamp": time.time(), "ip": request.remote_addr}
    Path(CHECK_IN_FILE).write_text(json.dumps(data))

    return jsonify({"status": "ok", "message": "Check-in recorded"}), 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

```bash
# Import hmac (fix the script):
# Add "import hmac" at the top of the Flask script

# Run with gunicorn behind nginx:
pip install flask gunicorn
gunicorn -b 127.0.0.1:5000 checkin_server:app
```

---

### Step 3: Encrypt the Payload

```bash
# Create your payload (credentials, instructions, etc.)
nano /tmp/dms-payload.txt
# Write what trusted contacts need to know

# Encrypt with their GPG public keys
gpg --import alice-public-key.asc
gpg --import bob-public-key.asc

gpg --encrypt \
  --recipient alice@example.com \
  --recipient bob@example.com \
  --output /var/dms/payload.enc \
  /tmp/dms-payload.txt

# Verify encryption
gpg --list-packets /var/dms/payload.enc

# Shred plaintext
shred -uz /tmp/dms-payload.txt
```

---

### Step 4: Crontab Setup

```bash
# Create DMS directory with restricted permissions
sudo mkdir -p /var/dms
sudo chown youruser:youruser /var/dms
chmod 700 /var/dms

# Add to crontab (check every 6 hours)
crontab -e
# Add:
0 */6 * * * /usr/bin/python3 /usr/local/bin/dms-check.py >> /var/log/dms.log 2>&1
```

---

### Step 5: Check-In from Anywhere

```bash
# Check in via curl (save this as a shell alias)
alias checkin='curl -s -X POST "https://yourserver.com/dms/checkin?token=YOUR_TOKEN"'

# Or from a phone, open:
# https://yourserver.com/dms/checkin?token=YOUR_TOKEN

# From anywhere with internet access:
curl "https://yourserver.com/dms/checkin?token=YOUR_TOKEN"
```

---

### Step 6: Reset After Release

If the DMS fires and you're fine:

```bash
# Update the check-in timestamp immediately
curl "https://yourserver.com/dms/checkin?token=YOUR_TOKEN"

# Reset the state file (clear "released" flags)
echo "{}" > /var/dms/state.json

# Contact trusted contacts to confirm you're OK
```

### Step 7: Hardening the Check-In Server

The simple Flask check-in server works for personal use, but a production deployment on an internet-facing server needs additional hardening to prevent the token from being brute-forced or the server from being disrupted.

**Rate limiting**: Add rate limiting to the check-in endpoint to prevent brute-force token guessing. With a 32-character random token, brute force is computationally infeasible, but rate limiting reduces noise in your logs and prevents your server from being used as part of a larger scanning campaign:

```python
from flask import Flask, request, jsonify, abort
from functools import wraps
import time
import hmac

app = Flask(__name__)

# Simple in-memory rate limiter (use Redis for multi-process deployments)
request_times = {}

def rate_limit(max_requests=5, window_seconds=60):
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            ip = request.remote_addr
            now = time.time()
            times = request_times.get(ip, [])
            # Remove old timestamps outside the window
            times = [t for t in times if now - t < window_seconds]
            if len(times) >= max_requests:
                abort(429)
            times.append(now)
            request_times[ip] = times
            return f(*args, **kwargs)
        return wrapper
    return decorator

@app.route("/dms/checkin", methods=["POST", "GET"])
@rate_limit(max_requests=10, window_seconds=60)
def checkin():
    token = request.args.get("token") or request.form.get("token")
    if not token:
        return jsonify({"error": "Missing token"}), 400
    if not hmac.compare_digest(token.encode(), VALID_TOKEN.encode()):
        time.sleep(1)  # Slow down invalid attempts
        return jsonify({"error": "Invalid token"}), 403
    # ... rest of check-in logic
```

**Nginx authentication layer**: Put nginx in front of the Flask server with an additional IP allowlist if you always check in from known locations:

```nginx
location /dms/checkin {
    # Only allow check-ins from your known IP ranges
    allow 203.0.113.0/24;    # Home IP range
    allow 198.51.100.5;      # Mobile carrier NAT exit
    deny all;

    proxy_pass http://127.0.0.1:5000;
    proxy_set_header X-Real-IP $remote_addr;
}
```

This provides defense-in-depth: even if your token is compromised, attackers from unknown IPs cannot use it to reset the DMS state.

### Step 8: Distributing DMS Across Multiple Servers

A single-server DMS has a significant failure mode: if that server goes down—due to payment lapse, infrastructure failure, or targeted attack—the DMS stops checking and either fires prematurely or fails to fire at all. For high-assurance use cases, distribute the check-in across multiple independent servers.

The simplest approach runs the same DMS script on two servers from different providers (e.g., Hetzner and Vultr). Both receive check-ins. The payload releases only when both servers agree the threshold has been exceeded:

```python
# Distributed check-in: sends the check-in token to multiple servers simultaneously
import requests
import concurrent.futures

DMS_ENDPOINTS = [
    "https://dms1.yourserver.com/dms/checkin",
    "https://dms2.backupserver.net/dms/checkin",
]
TOKEN = "YOUR_SECRET_TOKEN"

def checkin_server(url):
    try:
        r = requests.post(url, params={"token": TOKEN}, timeout=10)
        return url, r.status_code == 200
    except Exception as e:
        return url, False

def checkin_all():
    with concurrent.futures.ThreadPoolExecutor() as executor:
        results = list(executor.map(checkin_server, DMS_ENDPOINTS))

    all_ok = all(ok for _, ok in results)
    for url, ok in results:
        status = "OK" if ok else "FAILED"
        print(f"{status}: {url}")

    if not all_ok:
        print("WARNING: Not all DMS servers received check-in")

    return all_ok

if __name__ == "__main__":
    checkin_all()
```

Run this script as your check-in command instead of a raw `curl` call. The console output tells you immediately if a server failed to receive the check-in, giving you time to investigate the issue before the DMS timer expires.

---

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Related Articles

- [Set Up a Dead Man's Switch Email That Sends Credentials If](/privacy-tools-guide/how-to-set-up-dead-mans-switch-email-that-sends-credentials-/)
- [Use Dead Man's Switch with Multiple Independent Trustees](/privacy-tools-guide/how-to-use-dead-mans-switch-with-multiple-independent-truste/)
- [Set Up Dead Man's Switch Using Cron Job to Release Encrypted](/privacy-tools-guide/how-to-set-up-dead-mans-switch-using-cron-job-to-release-enc/)
- [How to Remove Personal Data from Data Brokers 2026:](/privacy-tools-guide/how-to-remove-personal-data-from-data-brokers-2026/---)
- [How to Remove Personal Data from Data](/privacy-tools-guide/how-to-remove-personal-data-from-data-brokers/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
