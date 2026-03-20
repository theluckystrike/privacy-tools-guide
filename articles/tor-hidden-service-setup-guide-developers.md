---
layout: default
title: "Tor Hidden Service Setup Guide Developers"
description: "A practical developer guide to setting up Tor hidden services. Learn configuration, security best practices, and deployment workflows for .onion services."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /tor-hidden-service-setup-guide-developers/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Tor hidden services (.onion addresses) allow you to host websites or APIs that are accessible only through the Tor network, with both your server location and users' identities hidden from passive observers because traffic routes through multiple relays in both directions. This guide shows you how to configure the Tor service with HiddenServicePort settings, generate your .onion address, harden security with network isolation, and integrate hidden services into your development workflow.

## Prerequisites

Before you begin, ensure you have access to a Linux server (Debian, Ubuntu, or Fedora are all well-supported) and the Tor software. You will need root or sudo access to configure the service. This guide assumes you are comfortable working from the command line and understand basic networking concepts.

## Installing Tor

Most Linux distributions include Tor in their default repositories. On Ubuntu or Debian:

```bash
sudo apt update
sudo apt install tor
```

On Fedora or RHEL-based systems:

```bash
sudo dnf install tor
```

Verify the installation by checking the Tor version:

```bash
tor --version
```

You should see output indicating Tor is installed. If you need a newer version, consider using the Tor Project's official repository.

## Configuring Your First Hidden Service

The Tor configuration file is located at `/etc/tor/torrc` on most Linux systems. Open this file with your preferred text editor:

```bash
sudo nano /etc/tor/torrc
```

Scroll down to the section labeled `## This section is for onion services`. You will find commented-out examples. Add the following configuration to create a basic hidden service:

```conf
HiddenServiceDir /var/lib/tor/my_hidden_service
HiddenServicePort 80 127.0.0.1:8080
```

Breaking this down:
- `HiddenServiceDir` specifies where Tor will store your service's private key and hostname
- `HiddenServicePort 80 127.0.0.1:8080` tells Tor to forward incoming requests on port 80 (the standard HTTP port) to your local web server running on port 8080

Save the file and restart Tor:

```bash
sudo systemctl restart tor
```

## Retrieving Your Onion Address

After Tor restarts, check the hidden service directory for the generated hostname:

```bash
sudo cat /var/lib/tor/my_hidden_service/hostname
```

This command returns a string ending in `.onion` — your service's public address. Share this address with users who have Tor Browser installed. They can access your service by entering the .onion address directly in the Tor Browser address bar.

## Running a Web Server for Your Hidden Service

Your hidden service needs a local web server to serve content. A simple Python HTTP server works well for testing:

```bash
# Run on port 8080 to match our Tor configuration
python3 -m http.server 8080 --directory /var/www/html
```

For production deployments, consider using nginx or Apache. Configure your web server to listen only on localhost (127.0.0.1) to prevent accidental exposure of your content outside the Tor network.

## Security Hardening for Hidden Services

Running a hidden service requires additional security considerations beyond a standard web server.

### Disable Directory Information Leakage

By default, Tor nodes can learn about your hidden service through directory requests. Add the following to your `torrc` to reduce this exposure:

```conf
HiddenServiceDir /var/lib/tor/my_hidden_service
HiddenServicePort 80 127.0.0.1:8080
HiddenServiceVersion 3
```

Using version 3 onion addresses provides improved cryptography and better resistance to enumeration attacks compared to version 2 addresses.

### Restrict Access by Client Authorization

For private services accessible only to authorized users, configure client authentication:

```conf
HiddenServiceDir /var/lib/tor/private_service
HiddenServicePort 80 127.0.0.1:8081
HiddenServiceVersion 3
HiddenServiceAuthorizeClient stealth myapp
```

This configuration creates a stealth authorization scheme named "myapp". After restarting Tor, retrieve the client authentication line from the hostname file:

```bash
sudo cat /var/lib/tor/private_service/hostname
```

The output includes an authentication string that authorized clients add to their Tor configuration. Users must configure this in their `torrc`:

```conf
ClientOnionAuthDir /var/lib/tor/authdir
```

Place the authentication file in the specified directory with the format `hostname:port=key`. This ensures only authorized clients can reach your service.

### Run Tor in a Container

For isolated deployments, run Tor inside a Docker container:

```dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y tor && rm -rf /var/lib/apt/lists/*
COPY torrc /etc/tor/torrc
RUN mkdir -p /var/lib/tor/hidden_service
RUN chown -R debian-tor:debian-tor /var/lib/tor
USER debian-tor
CMD ["tor"]
```

Build and run the container:

```bash
docker build -t tor-hidden-service .
docker run -d -p 127.0.0.1:9050:9050 --name my-tor-service tor-hidden-service
```

This approach keeps your Tor installation isolated from your host system.

## Automating Deployment with Systemd

Create a systemd service to manage your web application alongside Tor:

```ini
[Unit]
Description=My Hidden Service Web App
After=network.target tor.service
Requires=tor.service

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/start.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

This ensures your application starts after Tor is ready. Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp
```

## Testing Your Hidden Service

From your local machine, install Tor Browser and navigate to your .onion address. Verify that:
- The page loads without errors
- Your server logs show the request
- No IP addresses outside the Tor network appear in your logs

You can also test from the command line using `torsocks`:

```bash
torsocks curl http://your-onion-address.onion
```

This confirms your service works for users connecting through Tor.

## Performance Considerations

Hidden services introduce latency because traffic bounces through multiple Tor nodes. To optimize:
- Keep your payloads small
- Enable HTTP keep-alive connections
- Consider static site generation over dynamic server-side rendering
- Monitor circuit build times and rotate to faster relays if needed

## Summary

Setting up a Tor hidden service involves installing Tor, configuring your `torrc` file, and ensuring your local web server is properly secured. For developers, the key steps are using version 3 onion addresses, restricting access through client authorization when needed, and running services in isolated environments. With your hidden service running, you can offer access to your application without revealing your server's location or exposing it to the broader internet.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
