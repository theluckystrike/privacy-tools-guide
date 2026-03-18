---
layout: default
title: "How to Configure Cloak Plugin for Shadowsocks to Evade Active Probing Detection"
description: "A practical guide for developers and power users on configuring the Cloak plugin with Shadowsocks to resist active probing detection and censorship."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-configure-cloak-plugin-for-shadowsocks-to-evade-activ/
categories: [guides, privacy, security]
---

{% raw %}

Active probing detection represents one of the most sophisticated censorship techniques deployed by authoritarian networks. Unlike simple port blocking, active probing involves censors sending crafted packets to your server to identify the protocol being used. Shadowsocks alone provides encryption but lacks built-in protection against this attack vector. The Cloak plugin solves this problem by implementing protocol obfuscation and authentication that makes your Shadowsocks server appear as normal HTTPS traffic.

This guide walks through installing and configuring the Cloak plugin to transform your Shadowsocks setup into something that resists active probing detection.

## Understanding Active Probing

When a censor performs active probing, they send packets designed to trigger specific responses from known circumvention protocols. Shadowsocks servers respond in ways that reveal the protocol, enabling censors to identify and block the connection even when encryption prevents content inspection. Cloak addresses this by multiplexing Shadowsocks traffic over a TLS tunnel that looks indistinguishable from regular HTTPS traffic.

The plugin operates at the transport layer, meaning your Shadowsocks configuration remains unchanged. Cloak handles the obfuscation transparently, creating a dual-layer architecture where the outer TLS connection resists classification while the inner Shadowsocks connection provides the actual proxy functionality.

## Installing Cloak

Cloak is written in Go and compiles to a single binary. The installation process varies by operating system, but the general approach involves downloading the compiled binary or building from source.

On Linux servers, download the appropriate binary:

```bash
# Download the latest release (check github.com/cbeuw/Cloak for current version)
VERSION="v2.2.1"
wget "https://github.com/cbeuw/Cloak/releases/download/${VERSION}/ck-server-linux-amd64"
chmod +x ck-server-linux-amd64
sudo mv ck-server-linux-amd64 /usr/local/bin/ck-server
```

On macOS, you can use Homebrew:

```bash
brew install cloak
```

Verify the installation:

```bash
ck-server -version
```

## Server-Side Configuration

Cloak requires a configuration file in JSON format. Create `/etc/cloak.json` on your server with the following structure:

```json
{
  "Listen": "0.0.0.0:443",
  "ProxyProtocol": "shadowsocks",
  "EncryptionMethod": "chacha20-poly1305",
  "UID": "YOUR_UID_HERE",
  "PublicKey": "YOUR_PUBLIC_KEY_HERE",
  "PrivateKey": "YOUR_PRIVATE_KEY_HERE",
  "AdminUID": "YOUR_ADMIN_UID_HERE",
  "DatabasePath": "/var/lib/cloak/users.json"
}
```

Generate the required cryptographic keys using the ck-server binary:

```bash
# Generate keys (run locally or use a key generation tool)
ck-server -keygen
```

This outputs your UID, public key, and private key. The UID serves as a user identifier, while the public and private keys handle authentication.

The `AdminUID` is a special UID that grants administrative access, allowing you to manage connected users without authentication. In production deployments, you typically run a separate instance for administration or use the management API.

Create the database file:

```bash
sudo mkdir -p /var/lib/cloak
sudo touch /var/lib/cloak/users.json
```

The database stores user credentials and connection statistics. Each user entry includes their UID, data transfer limits, and connection parameters.

## Configuring Shadowsocks with Cloak

Your Shadowsocks configuration needs modification to work with Cloak. Edit your Shadowsocks server configuration (typically in `/etc/shadowsocks-libev/config.json`):

```json
{
  "server": "127.0.0.1",
  "server_port": 8443,
  "password": "your_shadowsocks_password",
  "method": "chacha20-ietf-poly1305",
  "mode": "tcp_only"
}
```

The key change is setting the server to localhost on port 8443. Shadowsocks now listens internally, while Cloak handles external connections and forwards them to Shadowsocks.

Restart both services:

```bash
# Restart Shadowsocks
sudo systemctl restart shadowsocks-libev

# Start Cloak
sudo ck-server -c /etc/cloak.json
```

Configure Cloak to start automatically:

```bash
sudo systemctl enable cloak
sudo systemctl start cloak
```

## Client-Side Configuration

On client machines, you need both a Shadowsocks client and the Cloak plugin. Install Cloak using the same method as the server, then configure your client.

Create a client configuration file `/etc/cloak-client.json`:

```json
{
  "RemoteHost": "your-server-ip",
  "RemotePort": 443,
  "LocalHost": "127.0.0.1",
  "LocalPort": 1080,
  "UID": "YOUR_UID_FROM_SERVER",
  "PublicKey": "YOUR_PUBLIC_KEY_FROM_SERVER",
  "PrivateKey": "YOUR_PRIVATE_KEY_FROM_SERVER",
  "ServerName": "your-domain.com",
  "NumConn": 4,
  "KeepAlive": 30
}
```

The `ServerName` field should match a valid TLS certificate on your server—this helps the connection pass through TLS inspection systems.

Configure your Shadowsocks client to use the Cloak plugin. In many clients, this means adding the plugin path and enabling it:

- Plugin: `ck-client`
- Plugin arguments: `-c /etc/cloak-client.json`

When the client connects, it performs a TLS handshake using the server's domain name. The connection appears identical to legitimate HTTPS traffic to external observers. The inner Shadowsocks protocol only activates after the TLS tunnel is established.

## Managing Users

Cloak includes a command-line tool for user management. Add a new user:

```bash
ck-server -uidgen | tee new_user.json
```

This generates a new UID and keys. Add the user to the database:

```bash
ck-server -adduser -c /etc/cloak.json -u NEW_UID -t 10000000 -d 500000000
```

This creates a user with a time limit of 10,000,000 seconds and a data limit of 500,000,000 bytes. Set limits to zero for unlimited access.

View connected users:

```bash
ck-server -status -c /etc/cloak.json
```

Remove a user:

```bash
ck-server -deluser -c /etc/cloak.json -u USER_UID
```

## Production Considerations

For production deployments, consider these additional measures:

**TLS Certificate**: Use a valid TLS certificate from Let's Encrypt. Cloak's TLS tunneling relies on proper certificate validation to establish trust.

```bash
sudo certbot certonly -d your-domain.com
sudo cp /etc/letsencrypt/live/your-domain.com/fullchain.pem /etc/cloak/tls.crt
sudo cp /etc/letsencrypt/live/your-domain.com/privkey.pem /etc/cloak/tls.key
```

**Obfuscated Servers**: Cloak works best when hosted on IP addresses that aren't already flagged. Consider using dedicated IPs for your Shadowsocks-Cloak setup rather than shared hosting.

**Firewall Configuration**: Ensure only port 443 is exposed externally. Shadowsocks on port 8443 should bind only to localhost.

## Troubleshooting

If connections fail, verify these common issues:

1. **Certificate validation**: Ensure the client has the correct time and can validate the server's TLS certificate
2. **Firewall rules**: Confirm port 443 accepts incoming connections
3. **Keys match**: Verify UID, public key, and private key are consistent between server and client
4. **Service status**: Check that both Cloak and Shadowsocks are running:

```bash
systemctl status cloak
systemctl status shadowsocks-libev
```

The logs often reveal specific failure points:

```bash
journalctl -u cloak -f
```

Cloak provides robust protection against active probing by transforming your Shadowsocks traffic into TLS-encrypted connections that blend with legitimate web traffic. This configuration requires more setup than basic Shadowsocks but offers significantly better resistance to sophisticated censorship infrastructure.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
