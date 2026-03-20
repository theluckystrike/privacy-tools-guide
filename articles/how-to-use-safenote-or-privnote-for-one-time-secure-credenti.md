---
layout: default
title: "How to Use SafeNote or PrivNote for One-Time Secure Credential Sharing"
description: "Learn how to securely share sensitive credentials one-time using SafeNote and PrivNote. Practical examples for developers and power users."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-use-safenote-or-privnote-for-one-time-secure-credenti/
---

{% raw %}

Sharing sensitive credentials securely is a common challenge for developers and system administrators. Whether you're sharing API keys, database passwords, or temporary access tokens, you need a method that ensures the recipient can view the information only once before it disappears forever. This is where one-time note services like SafeNote and PrivNote come into play.

## Understanding One-Time Secure Notes

One-time note services work on a simple but powerful principle: the secret is stored, a unique link is generated, and after the link is visited once, the content is permanently deleted from the server. This eliminates the risk of the credential lingering in inboxes, chat logs, or server logs where it could be discovered later.

For developers working with executors, CI/CD pipelines, or automated deployment systems, these tools provide a secure way to pass sensitive values without hardcoding them or storing them in configuration files that might be committed to version control.

## SafeNote: Open-Source Self-Hosting Option

SafeNote offers an open-source alternative that you can self-host, giving you complete control over where your sensitive data resides. This makes it particularly attractive for organizations with strict data sovereignty requirements.

### Setting Up SafeNote

If you prefer self-hosting, you can deploy SafeNote using Docker:

```bash
docker run -d -p 8080:5000 --name safenote \
  -e SECRET_KEY=your-secure-random-key \
  safenote/safenote:latest
```

Once running, you can create a one-time note programmatically using the API:

```bash
curl -X POST http://localhost:8080/api/notes \
  -H "Content-Type: application/json" \
  -d '{
    "content": "DB_PASSWORD=super-secret-password-123",
    "burn_after_reading": true,
    "expiration": "1h"
  }'
```

The response returns a unique URL that you can share:

```json
{
  "url": "http://localhost:8080/note/abc123xyz",
  "expiration": "2026-03-16T14:30:00Z"
}
```

### Using SafeNote with Executors

When working with executors or automation tools, you can integrate SafeNote into your workflows:

```python
import requests
import os

def create_secure_note(content, expiry_hours=1):
    """Create a one-time secure note via SafeNote API."""
    response = requests.post(
        f"{os.environ['SAFENOTE_URL']}/api/notes",
        json={
            "content": content,
            "burn_after_reading": True,
            "expiration": f"{expiry_hours}h"
        },
        headers={
            "Authorization": f"Bearer {os.environ['SAFENOTE_API_KEY']}"
        }
    )
    return response.json()["url"]

# Share database credentials with a deployment executor
db_creds = "POSTGRES_PASSWORD=prod-db-secret-key"
note_url = create_secure_note(db_creds, expiry_hours=2)
print(f"Share this URL with your executor: {note_url}")
```

The executor can then retrieve the note programmatically:

```python
def retrieve_and_burn_note(note_url):
    """Retrieve a one-time note and burn it immediately."""
    response = requests.get(note_url)
    if response.status_code == 200:
        # Note is now destroyed after this request
        return response.json()["content"]
    elif response.status_code == 404:
        return "Note already burned or does not exist"
    else:
        raise Exception(f"Failed to retrieve note: {response.text}")

secret = retrieve_and_burn_note(note_url)
# Use the secret for your task, then it's gone forever
```

## PrivNote: Quick Web-Based Solution

PrivNote provides a hosted service that requires no setup. You visit the website, create a note, and share the generated link. It's ideal for quick, ad-hoc credential sharing without any infrastructure overhead.

### Using PrivNote via the Web Interface

1. Navigate to privnote.com
2. Enter your sensitive credential in the text area
3. Enable "Burn after reading" for one-time access
4. Set an optional expiration time (1 hour to 30 days)
5. Click "Create Note" and copy the generated link
6. Send the link to your recipient

### PrivNote API Integration

For developers who need programmatic access, PrivNote offers an API:

```python
import requests
from requests.auth import HTTPBasicAuth

def create_privnote(secret_message, burn_on_read=True):
    """Create a PrivNote via API."""
    payload = {
        "data": secret_message,
        "burn_on_read": "1" if burn_on_read else "0",
        "expiration": "7d"  # 7 days max for free tier
    }
    
    response = requests.post(
        "https://privnote.com/api/v1/create",
        data=payload,
        auth=HTTPBasicAuth("your-api-key", "")
    )
    
    result = response.json()
    return {
        "link": result["note_link"],
        "destroy_key": result["destroy_key"]  # Keep this for manual deletion
    }

# Generate a one-time credential share link
credential_note = create_privnote(
    "AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE\nAWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
)
print(f"Share this: {credential_note['link']}")
print(f"Destroy key (save if needed): {credential_note['destroy_key']}")
```

## Security Considerations for Developers

While one-time notes are useful, you should understand their limitations and follow security best practices.

### What One-Time Notes Protect Against

- Credentials persisting in email inboxes
- Sensitive data appearing in chat logs
- Passwords stored in password managers that sync across devices
- Configuration files accidentally committed to repositories

### What One-Time Notes Don't Protect Against

- Man-in-the-middle attacks if you're not using HTTPS
- Recipients who screenshot the content before it burns
- Server-side compromises if using hosted services
- Timing attacks where someone guesses the link

### Best Practices

Always verify you're using HTTPS when creating or viewing notes. For self-hosted solutions like SafeNote, implement proper TLS certificates using Let's Encrypt or your organization's PKI.

When sharing credentials with executors, consider combining one-time notes with additional verification:

```python
import hashlib
import secrets

def create_verified_note_link(base_url, secret, verification_code):
    """Create a note link that requires separate verification."""
    note_data = f"{secret}\n\nVerification: {verification_code}"
    
    response = requests.post(
        f"{base_url}/api/notes",
        json={
            "content": note_data,
            "burn_after_reading": True
        }
    )
    
    note_link = response.json()["url"]
    return f"{note_link}?code={verification_code}"

# Share the link and verification code through different channels
verification_code = secrets.token_hex(8)
link = create_verified_note_link("https://your-safenote-instance.com", "API_KEY=xxx", verification_code)
# Send link via chat, verification code via SMS
```

This layered approach ensures that even if someone intercepts the note link, they cannot access the content without the separate verification code delivered through a different channel.

## Comparing SafeNote vs PrivNote

For developers and power users, the choice depends on your requirements:

| Feature | SafeNote (Self-Hosted) | PrivNote (Hosted) |
|---------|------------------------|-------------------|
| Data control | Full ownership | Third-party server |
| Setup required | Yes (Docker/server) | No |
| API access | Built-in | API key required |
| Customization | Full source access | Limited |
| Cost | Server costs | Free tier / Premium |

Self-hosting SafeNote gives you transparency over how your data is handled, while PrivNote offers convenience for quick, ad-hoc sharing without infrastructure management.

## Conclusion

One-time secure note services provide a practical solution for sharing sensitive credentials safely. Whether you choose the self-hosted flexibility of SafeNote or the quick convenience of PrivNote, the fundamental principle remains the same: the recipient gets one chance to view the information, then it's gone forever.

For developers integrating these tools into automated workflows, the APIs enable programmatic note creation and retrieval, making them suitable for CI/CD pipelines, deployment scripts, and executor-based task automation. Just remember that these tools supplement—not replace—proper secret management infrastructure for production environments.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
