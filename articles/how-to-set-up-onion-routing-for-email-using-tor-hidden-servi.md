---

layout: default
title: "How to Set Up Onion Routing for Email Using Tor Hidden Services"
description: "A practical guide for developers and power users to route email through Tor hidden services for enhanced privacy and anonymity."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /how-to-set-up-onion-routing-for-email-using-tor-hidden-servi/
categories: [security, guides]
reviewed: true
score: 8
---

{% raw %}

Onion routing through Tor provides a powerful mechanism for obscuring email communication metadata. While end-to-end encryption protects message content, network-level surveillance can still reveal who communicates with whom and when. By routing email through Tor hidden services, you add a significant layer of protection against traffic analysis.

This guide walks through setting up email delivery through Tor onion services. You'll configure a mail server with an onion address and set up your email client to route outbound mail through the Tor network.

## Understanding Onion Email Architecture

Traditional email travels in plaintext across multiple hops between mail servers. Even with TLS, the headers reveal sender, recipient, timestamps, and server paths. Tor hidden services replace this exposure with a unique `.onion` address that provides end-to-end encryption between clients and servers while masking the server's IP address.

The architecture involves three components: a Tor-enabled mail server running as a hidden service, client configuration to connect via SOCKS proxy, and proper key management for authentication.

## Prerequisites and Initial Setup

Before beginning, ensure you have a working Tor installation. On Linux, install it via your package manager:

```bash
sudo apt-get install tor
```

On macOS, use Homebrew:

```bash
brew install tor
```

Verify the installation:

```bash
tor --version
```

You'll also need a mail transfer agent. Postfix is a solid choice for this setup due to its flexibility and Tor integration capabilities.

## Configuring Tor as a Hidden Service

Edit your Tor configuration file to enable hidden service functionality for SMTP:

```bash
sudo nano /etc/tor/torrc
```

Add the following configuration to create an SMTP hidden service:

```
HiddenServiceDir /var/lib/tor/mail_hidden_service
HiddenServicePort 25 127.0.0.1:25
HiddenServicePort 587 127.0.0.1:587
```

The `HiddenServiceDir` specifies where Tor stores the onion address private key. The `HiddenServicePort` directives map external onion ports to your local mail server ports.

Create the directory and set proper permissions:

```bash
sudo mkdir -p /var/lib/tor/mail_hidden_service
sudo chown debian-tor:debian-tor /var/lib/tor/mail_hidden_service
sudo chmod 700 /var/lib/tor/mail_hidden_service
```

Restart Tor to generate your onion address:

```bash
sudo systemctl restart tor
```

Retrieve your new onion address:

```bash
sudo cat /var/lib/tor/mail_hidden_service/hostname
```

This command outputs something like `abcdef1234567890.onion`—share this address with correspondents who want to reach you via your hidden service.

## Setting Up Postfix for Tor Integration

Configure Postfix to listen on localhost and work with Tor. Edit your Postfix configuration:

```bash
sudo nano /etc/postfix/main.cf
```

Add or modify these settings:

```
inet_interfaces = 127.0.0.1
inet_protocols = ipv4
smtp_bind_address = 127.0.0.1
```

This ensures Postfix only accepts local connections, which is appropriate when Tor handles external access.

Configure Postfix to route outbound mail through Tor's SOCKS proxy by adding a transport map:

```
transport_maps = hash:/etc/postfix/transport
socks_proxy = socks5h://localhost:9050
```

Create the transport file:

```bash
sudo nano /etc/postfix/transport
```

Add domains you want to route through Tor:

```
gmail.com     smtp:[gmail-smtp-in.l.google.com]:25
yahoo.com     smtp:[mta5.yahoo.com]:25
*             smtp:[mail-relay.onion]:25
```

Generate the hash database:

```bash
sudo postmap /etc/postfix/transport
```

Restart Postfix:

```bash
sudo systemctl restart postfix
```

## Client Configuration for Onion Email

Configure your email client to route mail through the Tor network. Most clients support SOCKS proxy configuration.

For Thunderbird, go to Account Settings, select your outgoing server, and configure:

- Server: your-onion-address.onion
- Port: 587
- Connection security: STARTTLS
- Authentication method: Normal password
- SOCKS Host: 127.0.0.1
- SOCKS Port: 9050
- SOCKS Version: 5

For command-line usage with curl or similar tools, route SMTP through Tor:

```bash
curl --socks5-hostname 127.0.0.1:9050 \
  --mail-from sender@yourdomain.onion \
  --mail-rcpt recipient@example.com \
  --upload-email email-body.txt \
  smtp://mail.onion-service.com:25/
```

## Verifying Your Setup

Test that your hidden service is reachable:

```bash
torify nc -zv your-onion-address.onion 25
```

If the connection succeeds, your hidden service is properly configured.

Verify that outbound mail routes through Tor by checking your public IP from within a test script:

```bash
curl --socks5-hostname 127.0.0.1:9050 ifconfig.me
```

This should return an IP address different from your regular internet connection, confirming Tor routing.

## Security Considerations

Onion routing for email significantly reduces metadata exposure but doesn't make you invincible. Consider these additional measures:

Key verification becomes critical since correspondents must trust your onion address. Use a dedicated keybase.io profile or publish your onion address signed with your PGP key.

Email headers still contain information your correspondent can observe. Strip unnecessary headers by configuring your mail client or adding header cleanup rules in Postfix.

Tor exit nodes can be blocked by some mail providers. You may need to maintain a regular SMTP fallback for delivery to restrictive providers.

Clock skew can deanonymize Tor connections. Ensure your system clock is accurate using NTP synchronization.

## Troubleshooting Common Issues

Connection timeouts often indicate Tor isn't running or the hidden service hasn't fully initialized. Check Tor's status:

```bash
sudo systemctl status tor
```

Authentication failures may stem from SOCKS proxy configuration. Ensure your client uses SOCKS Version 5 with hostname resolution enabled (`socks5h://` rather than `socks5://`).

Delivery failures to specific domains usually mean the recipient's mail server blocks Tor exit nodes. Check mail logs:

```bash
sudo tail -f /var/log/mail.log
```

## Performance Implications

Expect increased latency when routing email through Tor. A typical SMTP handshake adds 2-5 seconds, though actual delivery time varies based on Tor network congestion. For high-volume email, consider batch sending during off-peak hours.

Bandwidth usage approximately doubles since your traffic traverses multiple Tor nodes. Monitor your connection if you have data caps.

## Getting Started

Start small by setting up a hidden service for receiving email. Configure Tor and Postfix as described, share your onion address with trusted contacts, and verify delivery. Once comfortable with the setup, add outbound routing for complete email privacy.

The initial configuration requires attention to detail, but the resulting protection against traffic analysis justifies the effort. Your email metadata becomes significantly harder to collect and correlate.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
