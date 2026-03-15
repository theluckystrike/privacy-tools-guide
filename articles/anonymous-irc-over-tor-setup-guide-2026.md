---
layout: default
title: "Anonymous IRC Over Tor Setup Guide 2026"
description: "A practical guide to configuring IRC over Tor for anonymous communications. Learn to set up Tor daemon, configure IRC clients, connect to onion-service networks, and implement best practices for privacy-conscious developers."
date: 2026-03-15
author: theluckystrike
permalink: /anonymous-irc-over-tor-setup-guide-2026/
categories: [guides]
---

{% raw %}

Internet Relay Chat (IRC) remains a vital communication protocol for developer communities, open-source projects, and privacy-focused discussions. However, standard IRC connections expose your IP address, making traffic analysis and deanonymization attacks straightforward. Running IRC over Tor adds multiple layers of obfuscation, hiding both your location and the content of your communications from network observers.

This guide covers setting up Tor, configuring IRC clients, connecting to Tor-hidden IRC networks, and implementing operational security practices for developers and power users who require stronger anonymity guarantees.

## Installing and Configuring the Tor Daemon

The foundation of anonymous IRC is a properly configured Tor installation. On Linux, install the Tor daemon:

```bash
# Debian/Ubuntu
sudo apt install tor

# macOS
brew install tor

# Verify Tor is running
tor --version
```

Configure Tor for IRC traffic by editing `/etc/tor/torrc` (Linux) or `~/Library/Application Support/Tor/torrc` (macOS):

```conf
# Enable SOCKS proxy on port 9050
SOCKSPort 9050

# Optional: Control port for scripting
ControlPort 9051
CookieAuthentication 1

# Exclude certain exits for IRC to improve connectivity
ExcludeExitNodes {us},{gb},{de},{fr}

# Set circuit build timeout for IRC
CircuitBuildTimeout 10
```

Restart Tor to apply changes:

```bash
sudo systemctl restart tor
```

Verify the SOCKS proxy is listening:

```bash
netstat -an | grep 9050
```

## IRC Client Configuration

Several IRC clients support Tor natively. This guide focuses on three popular options: HexChat (GUI), WeeChat (terminal), and irssi (terminal).

### HexChat Configuration

HexChat provides a straightforward GUI for Tor connections:

1. Open HexChat → Network List → Add a new network
2. Set the server hostname to a Tor-based IRC network (e.g., `irc.oftc.net.onion`)
3. Navigate to the server and enable "Proxy" with these settings:
   - Proxy type: SOCKS5
   - Proxy host: 127.0.0.1
   - Proxy port: 9050

For OFTC, which maintains an official Tor hidden service:

```conf
# Server settings
Server/Hostname: irc6.oftc.net.onion
Port: 6697
Use SSL: Yes
Proxy: 127.0.0.1:9050 (SOCKS5)
```

### WeeChat Configuration

WeeChat is highly scriptable and works well for advanced users. Configure Tor support:

```bash
# Set up the relay plugin
/set relay.network.socks_proxy "127.0.0.1:9050"

# Add OFTC with Tor
/server add oftc-tor irc6.oftc.net.onion/6697 -ssl
/set irc.server.oftc-tor.proxy "socks5://127.0.0.1:9050"
/set irc.server.oftc-tor.autoconnect on
/connect oftc-tor
```

Verify the connection shows a .onion address:

```bash
/server
```

### Irssi Configuration

For minimal resource usage, configure irssi with Tor:

```perl
# In irassi, run:
/network add -socks_proxy 127.0.0.1 9050 oftc
/server add -net oftc -hostname irc6.oftc.net.onion -port 6697 -ssl
```

## Connecting to Tor-Based IRC Networks

Several IRC networks operate Tor hidden services, providing inherent protection against IP-based attacks.

### OFTC (Open and Free Technology Community)

OFTC hosts numerous Linux distribution and open-source project channels:

```bash
# WeeChat connection
/server add oftc irc6.oftc.net.onion/6697 -ssl
/set irc.server.oftc.proxy "socks5://127.0.0.1:9050"
/connect oftc
```

### IRCnet Tor Bridge

IRCnet provides a Tor bridge for users who cannot connect via regular IRC:

```conf
# torrc configuration
HiddenServiceDir /var/lib/tor/ircd/
HiddenServicePort 6667 irc.xxxiybernoobxxxi.onion:6667
```

### Custom Server Connections

Many networks allow connections through Tor exits. Use caution—exit nodes can be monitored:

```bash
# WeeChat: Connect through Tor exit nodes
/set irc.server.example.proxy "socks5://127.0.0.1:9050"
/server add example irc.example.org/6697 -ssl
```

## Verifying Anonymity

After connecting, verify your anonymity:

```bash
# In WeeChat, check your hostmask
/whois yournick

# The response should show the Tor exit node IP, not your real IP
# Example: yournick!~user@<random>.tor-exit.node.net
```

Test for DNS leaks:

```bash
# Check what DNS resolver was used
/dns yournick
```

## Operational Security Best Practices

Running IRC over Tor requires additional operational practices:

### Nickname Consistency

Avoid changing nicknames frequently—correlation attacks can link your identities. Choose a single pseudonym for all Tor-based IRC activity:

```weechat
/set irc.server.default.nicks "your_persistent_nick"
```

### TLS Certificate Pinning

Verify server certificates to prevent man-in-the-middle attacks:

```bash
# WeeChat: Display SSL certificate fingerprint
/set irc.server.oftc.tls_verify on
/save
```

### Automatic Reconnection

Configure reliable reconnection to maintain anonymity during network disruptions:

```weechat
/set irc.server.oftc.autoreconnect on
/set irc.server.oftc.autoreconnect_delay 30
```

### Logging Considerations

Disable logging of sensitive channels, or encrypt logs:

```weechat
/set logger.level.irc 0
/set logger.mask.irc "$plugin/$channel.isc.log"
```

## Troubleshooting Common Issues

Connection failures often stem from Tor circuit issues:

```bash
# Check Tor circuit status
torctl status

# Force new circuits
killall -HUP tor

# Increase timeout in torrc
CircuitBuildTimeout 30
```

SSL certificate errors typically indicate Tor exit node issues:

```weechat
# Temporarily disable SSL verification (use sparingly)
/set irc.server.oftc.tls_verify off
```

## Advanced: Running Your Own Tor-Enabled IRC Server

For complete control, run an IRC daemon with a Tor hidden service:

```conf
# torrc configuration
HiddenServiceDir /var/lib/tor/irc_server/
HiddenServicePort 6667 127.0.0.1:6667

# ircd.conf (inspircd example)
<bind address="127.0.0.1" port="6667">
```

This allows users to connect directly to your .onion address, eliminating exit node exposure entirely.

The setup requires a Tor daemon, a compatible IRC client, and attention to operational security practices. Start with a trusted network like OFTC, verify your anonymity through hostmask checks, and add advanced configurations as your requirements grow.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
