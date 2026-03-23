---
layout: default
title: "Secure Webhook Implementation Guide"
description: "Implement tamper-proof webhooks with HMAC signatures, replay protection, secret rotation, and IP allowlisting in Python and Node.js for production use"
date: 2026-03-22
author: theluckystrike
permalink: /secure-webhook-implementation-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
# Secure Webhook Implementation Guide

Webhooks are HTTP callbacks — when an event occurs on a third-party service, it sends a POST request to your endpoint. Without signature verification, anyone can send a fake webhook to trigger actions in your system. This guide covers the full stack of webhook security: HMAC signing, replay attack prevention, secret rotation, and delivery reliability.

## The Threat Model

- **Forged payloads**: An attacker crafts a payload that looks legitimate and POSTs it to your endpoint
- **Replay attacks**: A legitimate webhook is captured and re-sent to trigger the same action twice
- **SSRF via webhook delivery**: A malicious service sends webhooks from unexpected IPs
- **Secret leakage**: Webhook secrets committed to git or logged in plaintext

---

## 1. Sender Side: Sign Every Webhook

The sender (your service or a third-party you control) computes a HMAC of the request body and includes it in a header. The receiver verifies it.

### Python (FastAPI webhook sender)

```python
import hmac
import hashlib
import time
import secrets
import httpx
from typing import Any

WEBHOOK_SECRET = "whsec_your_256bit_secret_here"   # from environment

def sign_webhook(payload_bytes: bytes, secret: str, timestamp: int) -> str:
    """
    Compute HMAC-SHA256 of timestamp + payload.
    Including timestamp prevents replays even if signature is leaked.
    """
    message = f"{timestamp}.".encode() + payload_bytes
    sig = hmac.new(secret.encode(), message, hashlib.sha256).hexdigest()
    return f"v1={sig}"

async def deliver_webhook(url: str, event: str, data: dict) -> bool:
    import json
    payload = json.dumps({"event": event, "data": data}).encode()
    ts = int(time.time())
    signature = sign_webhook(payload, WEBHOOK_SECRET, ts)

    headers = {
        "Content-Type": "application/json",
        "X-Webhook-Signature": signature,
        "X-Webhook-Timestamp": str(ts),
        "X-Webhook-ID": secrets.token_hex(16),   # unique delivery ID
    }

    async with httpx.AsyncClient(timeout=10.0) as client:
        for attempt in range(3):
            try:
                resp = await client.post(url, content=payload, headers=headers)
                if resp.status_code < 500:
                    return True
            except httpx.TimeoutException:
                pass
            await asyncio.sleep(2 ** attempt)   # exponential backoff
    return False
```

---

## 2. Receiver Side: Verify Before Processing

### Python (FastAPI webhook receiver)

```python
import hmac
import hashlib
import time
import redis
from fastapi import FastAPI, Request, HTTPException, Header
from typing import Optional

app = FastAPI()
r = redis.Redis(host="127.0.0.1", port=6379, db=1)

WEBHOOK_SECRET    = "whsec_your_256bit_secret_here"
TIMESTAMP_TOLERANCE = 300   # reject webhooks older than 5 minutes

def verify_webhook(body: bytes, signature: str, timestamp: str,
                   webhook_id: Optional[str] = None) -> bool:
    # 1. Parse timestamp
    try:
        ts = int(timestamp)
    except (ValueError, TypeError):
        return False

    # 2. Check timestamp freshness (replay protection)
    if abs(time.time() - ts) > TIMESTAMP_TOLERANCE:
        return False

    # 3. Deduplicate by webhook ID (catch replays within tolerance window)
    if webhook_id:
        key = f"wh_seen:{webhook_id}"
        if r.exists(key):
            return False   # already processed
        r.setex(key, TIMESTAMP_TOLERANCE * 2, "1")

    # 4. Recompute expected signature
    message    = f"{ts}.".encode() + body
    expected   = hmac.new(
        WEBHOOK_SECRET.encode(), message, hashlib.sha256
    ).hexdigest()
    expected_header = f"v1={expected}"

    # 5. Compare using constant-time comparison (prevents timing attacks)
    return hmac.compare_digest(expected_header, signature)

@app.post("/webhooks/events")
async def receive_webhook(
    request: Request,
    x_webhook_signature: str = Header(...),
    x_webhook_timestamp:  str = Header(...),
    x_webhook_id: Optional[str] = Header(None),
):
    body = await request.body()

    if not verify_webhook(body, x_webhook_signature,
                          x_webhook_timestamp, x_webhook_id):
        raise HTTPException(status_code=401, detail="Invalid webhook signature")

    import json
    payload = json.loads(body)
    event   = payload.get("event")

    # Route to handler
    await handle_event(event, payload.get("data", {}))

    return {"status": "ok"}
```

### Node.js (Express webhook receiver)

```javascript
const express  = require('express');
const crypto   = require('crypto');
const redis    = require('redis');

const app    = express();
const client = redis.createClient();
await client.connect();

const WEBHOOK_SECRET      = process.env.WEBHOOK_SECRET;
const TIMESTAMP_TOLERANCE = 300 * 1000;   // 5 minutes in ms

// IMPORTANT: parse body as raw buffer, not JSON — JSON.stringify is not idempotent
app.use('/webhooks', express.raw({ type: 'application/json' }));

function verifyWebhook(body, signature, timestamp, webhookId) {
  const ts = parseInt(timestamp, 10);
  if (isNaN(ts)) return false;

  if (Math.abs(Date.now() - ts * 1000) > TIMESTAMP_TOLERANCE) {
    return false;   // stale
  }

  const message  = Buffer.concat([Buffer.from(`${ts}.`), body]);
  const expected = 'v1=' + crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(message)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}

app.post('/webhooks/events', async (req, res) => {
  const sig       = req.headers['x-webhook-signature'];
  const ts        = req.headers['x-webhook-timestamp'];
  const webhookId = req.headers['x-webhook-id'];

  // Deduplicate
  if (webhookId) {
    const seen = await client.get(`wh_seen:${webhookId}`);
    if (seen) {
      return res.status(200).json({ status: 'duplicate' });
    }
    await client.setEx(`wh_seen:${webhookId}`, 600, '1');
  }

  if (!verifyWebhook(req.body, sig, ts, webhookId)) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  const payload = JSON.parse(req.body);
  await handleEvent(payload.event, payload.data);

  res.json({ status: 'ok' });
});
```

---

## 3. IP Allowlisting

Most webhook providers publish their source IP ranges. Allowlist them at the firewall level:

```python
# FastAPI middleware — check source IP before signature verification
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

# Stripe webhook IPs (example — always fetch current list from provider)
ALLOWED_WEBHOOK_IPS = {
    "3.18.12.63", "3.130.192.231", "13.235.14.237",
    # ... fetch from https://stripe.com/files/ips/ips_webhooks.txt
}

class WebhookIPFilter(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        if request.url.path.startswith("/webhooks"):
            client_ip = request.client.host
            # Handle X-Forwarded-For if behind a load balancer
            forwarded = request.headers.get("X-Forwarded-For")
            if forwarded:
                client_ip = forwarded.split(",")[0].strip()
            if client_ip not in ALLOWED_WEBHOOK_IPS:
                return Response("Forbidden", status_code=403)
        return await call_next(request)

app.add_middleware(WebhookIPFilter)
```

---

## 4. Secret Rotation

Rotate webhook secrets without downtime by accepting both old and new secrets during a transition window:

```python
import os

WEBHOOK_SECRETS = [
    os.environ["WEBHOOK_SECRET_NEW"],
    os.environ["WEBHOOK_SECRET_OLD"],   # retired after rotation completes
]

def verify_any_secret(body: bytes, signature: str, timestamp: str) -> bool:
    """Try each secret in sequence — allows overlap during rotation."""
    for secret in WEBHOOK_SECRETS:
        ts = int(timestamp)
        message  = f"{ts}.".encode() + body
        expected = "v1=" + hmac.new(
            secret.encode(), message, hashlib.sha256
        ).hexdigest()
        if hmac.compare_digest(expected, signature):
            return True
    return False
```

Rotation procedure:
1. Generate new secret, add as `WEBHOOK_SECRET_NEW`; keep old as `WEBHOOK_SECRET_OLD`
2. Update webhook URL config on the sender side with new secret
3. After all in-flight webhooks have been delivered (typically 30–60 min), remove `WEBHOOK_SECRET_OLD`

---

## 5. Webhook Delivery Reliability

```python
import asyncio
from dataclasses import dataclass
from enum import Enum

class DeliveryStatus(Enum):
    PENDING   = "pending"
    DELIVERED = "delivered"
    FAILED    = "failed"

@dataclass
class WebhookDelivery:
    id:       str
    url:      str
    payload:  bytes
    attempts: int = 0
    status:   DeliveryStatus = DeliveryStatus.PENDING

async def delivery_worker(queue: asyncio.Queue):
    """Worker that processes delivery queue with exponential backoff."""
    while True:
        delivery: WebhookDelivery = await queue.get()
        ts  = int(time.time())
        sig = sign_webhook(delivery.payload, WEBHOOK_SECRET, ts)

        headers = {
            "X-Webhook-Signature": sig,
            "X-Webhook-Timestamp": str(ts),
            "X-Webhook-ID":        delivery.id,
            "Content-Type":        "application/json",
        }

        try:
            async with httpx.AsyncClient(timeout=10.0) as client:
                resp = await client.post(
                    delivery.url, content=delivery.payload, headers=headers
                )
            if 200 <= resp.status_code < 300:
                delivery.status = DeliveryStatus.DELIVERED
                return
        except Exception:
            pass

        delivery.attempts += 1
        if delivery.attempts < 5:
            await asyncio.sleep(min(30 * (2 ** delivery.attempts), 3600))
            await queue.put(delivery)
        else:
            delivery.status = DeliveryStatus.FAILED
            # Alert on permanent failure
```

---

## Security Checklist

- [ ] HMAC-SHA256 signature on every delivery
- [ ] Timestamp in signed payload; reject messages older than 5 minutes
- [ ] Unique webhook ID per delivery; deduplicated with Redis/cache
- [ ] `hmac.compare_digest` / `timingSafeEqual` for constant-time comparison
- [ ] Raw body buffer used for verification (not re-serialized JSON)
- [ ] IP allowlist at firewall or middleware layer
- [ ] Webhook secret stored in secret manager, not hardcoded
- [ ] Secret rotation supported with dual-secret transition window
- [ ] Retry queue with exponential backoff; alert on 5 consecutive failures

---

## Related Reading

- [Secure JWT Implementation Best Practices](/secure-jwt-implementation-best-practices/)
- [Secure API Gateway Setup with Kong](/kong-api-gateway-secure-setup-guide/)
- [Secure Environment Variable Management](/secure-environment-variable-management-guide/)
- [Best Password Manager with Secure Notes: A Technical Guide](/best-password-manager-with-secure-notes/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
