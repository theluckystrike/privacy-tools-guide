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

## Core DMS Script

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

## Check-In Server (Simple HTTP Endpoint)

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

## Encrypt the Payload

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

## Crontab Setup

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

## Check-In from Anywhere

```bash
# Check in via curl (save this as a shell alias)
alias checkin='curl -s -X POST "https://yourserver.com/dms/checkin?token=YOUR_TOKEN"'

# Or from a phone, open:
# https://yourserver.com/dms/checkin?token=YOUR_TOKEN

# From anywhere with internet access:
curl "https://yourserver.com/dms/checkin?token=YOUR_TOKEN"
```

---

## Reset After Release

If the DMS fires and you're fine:

```bash
# Update the check-in timestamp immediately
curl "https://yourserver.com/dms/checkin?token=YOUR_TOKEN"

# Reset the state file (clear "released" flags)
echo "{}" > /var/dms/state.json

# Contact trusted contacts to confirm you're OK
```

---

## Related Reading

- [How to Set Up Dead Man's Switch Email That Sends Credentials](/privacy-tools-guide/how-to-set-up-dead-mans-switch-email-that-sends-credentials-/)
- [How to Set Up Dead Man's Switch Using Cron Job](/privacy-tools-guide/how-to-set-up-dead-mans-switch-using-cron-job-to-release-enc/)
- [LUKS Encrypted Container Guide](/privacy-tools-guide/luks-encrypted-container-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
