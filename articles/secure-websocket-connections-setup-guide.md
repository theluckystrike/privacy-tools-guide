---
layout: default
title: "Secure WebSocket Connections Setup Guide"
description: "Configure WSS with TLS termination, origin validation, authentication tokens, and rate limiting to protect WebSocket endpoints in production"
date: 2026-03-22
author: theluckystrike
permalink: /secure-websocket-connections-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

WebSocket connections stay open and bypass the stateless request-response model that most HTTP security controls assume. An improperly secured WebSocket endpoint can be hijacked, used for cross-site WebSocket hijacking (CSWSH), or abused to exfiltrate data. This guide covers TLS, authentication, origin checks, and rate limiting for production WebSocket deployments.

## The Risks

**Unencrypted WebSockets (ws://)**: All messages are transmitted in plaintext, visible to anyone on the path between client and server.

**Cross-Site WebSocket Hijacking**: A malicious page can open a WebSocket to your server using the victim's cookies, since browsers send credentials automatically on WebSocket upgrade requests.

**Missing rate limits**: WebSocket connections are cheap to open and can be used to exhaust server resources or brute-force application protocols.

## Step 1: Always Use WSS (WebSocket Secure)

`wss://` is WebSocket over TLS — equivalent to HTTPS. Never use `ws://` in production.

In Nginx, configure the WebSocket proxy and TLS termination:

```nginx
# /etc/nginx/sites-available/myapp

server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;

    # WebSocket proxy
    location /ws {
        proxy_pass http://127.0.0.1:8765;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    # Redirect HTTP to HTTPS
    listen 80;
    return 301 https://$host$request_uri;
}
```

With Caddy (automatic TLS):
```
app.example.com {
    handle /ws* {
        reverse_proxy localhost:8765 {
            header_up Upgrade {http.upgrade}
            header_up Connection "upgrade"
        }
    }
}
```

## Step 2: Validate the Origin Header

The WebSocket handshake sends an `Origin` header. Unlike CORS, the browser does not block the connection if the server does not check it — so you must check it yourself:

```python
# Python websockets library example
import websockets
import asyncio

ALLOWED_ORIGINS = {"https://app.example.com", "https://www.example.com"}

async def handler(websocket, path):
    origin = websocket.request_headers.get("Origin", "")
    if origin not in ALLOWED_ORIGINS:
        await websocket.close(1008, "Invalid origin")
        return
    # Handle the connection...
    await handle_connection(websocket, path)

start_server = websockets.serve(
    handler,
    "127.0.0.1",
    8765,
    origins=list(ALLOWED_ORIGINS)
)

asyncio.get_event_loop().run_until_complete(start_server)
asyncio.get_event_loop().run_forever()
```

In Node.js with the `ws` library:

```javascript
const WebSocket = require('ws');

const ALLOWED_ORIGINS = new Set([
  'https://app.example.com',
  'https://www.example.com'
]);

const wss = new WebSocket.Server({
  port: 8765,
  verifyClient: (info, callback) => {
    const origin = info.origin;
    if (!ALLOWED_ORIGINS.has(origin)) {
      callback(false, 403, 'Forbidden: invalid origin');
      return;
    }
    callback(true);
  }
});

wss.on('connection', (ws, req) => {
  // Handle authenticated connection
  console.log(`New connection from ${req.socket.remoteAddress}`);
});
```

## Step 3: Authenticate WebSocket Connections

Do not rely on cookies alone — use tokens passed in the URL or as the first message:

### Token in URL Query Parameter

```javascript
// Client-side
const token = await getJWT(); // From your auth system
const ws = new WebSocket(`wss://app.example.com/ws?token=${token}`);

// Server-side (Node.js)
const url = require('url');
const jwt = require('jsonwebtoken');

wss.on('connection', (ws, req) => {
  const params = new url.URLSearchParams(url.parse(req.url).query);
  const token = params.get('token');

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    ws.user = decoded;
  } catch (err) {
    ws.close(4001, 'Unauthorized');
    return;
  }

  // Proceed with authenticated connection
});
```

Note: URL parameters appear in server logs. If log privacy matters, use the first-message pattern instead.

### Token in First Message

```javascript
// Server-side: require authentication as first message
wss.on('connection', (ws) => {
  ws.authenticated = false;
  ws.authTimeout = setTimeout(() => {
    if (!ws.authenticated) {
      ws.close(4001, 'Authentication timeout');
    }
  }, 5000); // 5 second window to authenticate

  ws.on('message', async (data) => {
    if (!ws.authenticated) {
      try {
        const { type, token } = JSON.parse(data);
        if (type !== 'auth') {
          ws.close(4001, 'Expected auth message');
          return;
        }
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        ws.user = decoded;
        ws.authenticated = true;
        clearTimeout(ws.authTimeout);
        ws.send(JSON.stringify({ type: 'auth_ok' }));
      } catch {
        ws.close(4001, 'Invalid token');
      }
      return;
    }
    // Handle regular messages after auth
    handleMessage(ws, data);
  });
});
```

## Step 4: Rate Limit Connections

```javascript
// Track connections per IP
const connectionCounts = new Map();
const MAX_CONNECTIONS_PER_IP = 10;
const RATE_WINDOW_MS = 60000; // 1 minute

wss = new WebSocket.Server({
  port: 8765,
  verifyClient: (info, callback) => {
    const ip = info.req.socket.remoteAddress;
    const now = Date.now();

    if (!connectionCounts.has(ip)) {
      connectionCounts.set(ip, { count: 0, resetAt: now + RATE_WINDOW_MS });
    }

    const record = connectionCounts.get(ip);
    if (now > record.resetAt) {
      record.count = 0;
      record.resetAt = now + RATE_WINDOW_MS;
    }

    record.count++;
    if (record.count > MAX_CONNECTIONS_PER_IP) {
      callback(false, 429, 'Too Many Connections');
      return;
    }

    callback(true);
  }
});
```

For Nginx-level rate limiting on the upgrade endpoint:

```nginx
# In nginx.conf http block
limit_req_zone $binary_remote_addr zone=ws_limit:10m rate=5r/s;

# In server block
location /ws {
    limit_req zone=ws_limit burst=10 nodelay;
    # ... proxy config
}
```

## Step 5: Message Validation

Treat every message as untrusted input:

```javascript
wss.on('connection', (ws) => {
  ws.on('message', (rawData) => {
    // Size limit
    if (rawData.length > 65536) { // 64KB max
      ws.close(1009, 'Message too large');
      return;
    }

    let msg;
    try {
      msg = JSON.parse(rawData);
    } catch {
      ws.close(1007, 'Invalid JSON');
      return;
    }

    // Validate message type
    const VALID_TYPES = new Set(['chat', 'ping', 'subscribe']);
    if (!VALID_TYPES.has(msg.type)) {
      ws.send(JSON.stringify({ error: 'Unknown message type' }));
      return;
    }

    // Process valid message
    handleMessage(ws, msg);
  });
});
```

## Step 6: Secure Headers for the Upgrade Request

Add security headers to the HTTP response that initiates the WebSocket upgrade:

```nginx
location /ws {
    add_header X-Content-Type-Options "nosniff" always;
    add_header Strict-Transport-Security "max-age=31536000" always;
    # No Content-Security-Policy needed here (it's not an HTML response)

    proxy_pass http://127.0.0.1:8765;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

## Related Reading

- [How to Set Up Caddy with Automatic HTTPS](/privacy-tools-guide/how-to-set-up-caddy-with-automatic-https/)
- [Secure Redis Deployment Without Exposure](/privacy-tools-guide/secure-redis-deployment-without-exposure/)
- [How to Configure UFW Firewall on Ubuntu](/privacy-tools-guide/how-to-configure-ufw-firewall-on-ubuntu/)

- [Secure Code Signing Setup for Developers](/privacy-tools-guide/secure-code-signing-setup-for-developers/)
- [AI Coding Assistant Session Data Lifecycle](https://theluckystrike.github.io/ai-tools-compared/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
---

## Related Articles

- [How To Use Outline Vpn Server For Creating Personal Proxy](/privacy-tools-guide/how-to-use-outline-vpn-server-for-creating-personal-proxy-in/)
- [How to Set Up a SOCKS5 Proxy with SSH](/privacy-tools-guide/socks5-proxy-ssh-setup-guide/)
- [How To Use Trojan Gfw Proxy To Disguise Traffic As Https](/privacy-tools-guide/how-to-use-trojan-gfw-proxy-to-disguise-traffic-as-https-fro/)
- [How to Secure PostgreSQL for Production](/privacy-tools-guide/secure-postgresql-production-guide/)
- [How to Self-Host Bitwarden Vaultwarden: Complete Setup Guide](/privacy-tools-guide/how-to-self-host-bitwarden-vaultwarden-complete-setup-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
