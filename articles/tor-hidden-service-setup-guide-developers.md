---
layout: default
title: "Tor Hidden Service Setup Guide Developers"
description: "A practical developer guide to setting up Tor hidden services. Learn configuration, security best practices, and deployment workflows for .onion services"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /tor-hidden-service-setup-guide-developers/
categories: [guides]
tags: [privacy-tools-guide]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Tor hidden services (.onion addresses) allow you to host websites or APIs that are accessible only through the Tor network, with both your server location and users' identities hidden from passive observers because traffic routes through multiple relays in both directions. This guide shows you how to configure the Tor service with HiddenServicePort settings, generate your .onion address, harden security with network isolation, and integrate hidden services into your development workflow.

Prerequisites

Before you begin, ensure you have access to a Linux server (Debian, Ubuntu, or Fedora are all well-supported) and the Tor software. You will need root or sudo access to configure the service. This guide assumes you are comfortable working from the command line and understand basic networking concepts.

Step 1: Install Tor

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

Step 2: Configure Your First Hidden Service

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

Step 3: Retrieve Your Onion Address

After Tor restarts, check the hidden service directory for the generated hostname:

```bash
sudo cat /var/lib/tor/my_hidden_service/hostname
```

This command returns a string ending in `.onion`. your service's public address. Share this address with users who have Tor Browser installed. They can access your service by entering the .onion address directly in the Tor Browser address bar.

Step 4: Run a Web Server for Your Hidden Service

Your hidden service needs a local web server to serve content. A simple Python HTTP server works well for testing:

```bash
Run on port 8080 to match our Tor configuration
python3 -m http.server 8080 --directory /var/www/html
```

For production deployments, consider using nginx or Apache. Configure your web server to listen only on localhost (127.0.0.1) to prevent accidental exposure of your content outside the Tor network.

Step 5: Security Hardening for Hidden Services

Running a hidden service requires additional security considerations beyond a standard web server.

Disable Directory Information Leakage

By default, Tor nodes can learn about your hidden service through directory requests. Add the following to your `torrc` to reduce this exposure:

```conf
HiddenServiceDir /var/lib/tor/my_hidden_service
HiddenServicePort 80 127.0.0.1:8080
HiddenServiceVersion 3
```

Using version 3 onion addresses provides improved cryptography and better resistance to enumeration attacks compared to version 2 addresses.

Restrict Access by Client Authorization

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

Run Tor in a Container

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

Step 6: Automate Deployment with Systemd

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

Step 7: Test Your Hidden Service

From your local machine, install Tor Browser and navigate to your .onion address. Verify that:
- The page loads without errors
- Your server logs show the request
- No IP addresses outside the Tor network appear in your logs

You can also test from the command line using `torsocks`:

```bash
torsocks curl http://your-onion-address.onion
```

This confirms your service works for users connecting through Tor.

Performance Considerations

Hidden services introduce latency because traffic bounces through multiple Tor nodes. To optimize:
- Keep your payloads small
- Enable HTTP keep-alive connections
- Consider static site generation over dynamic server-side rendering
- Monitor circuit build times and rotate to faster relays if needed

Troubleshooting

Configuration changes not taking effect

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

Permission denied errors

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

Connection or network-related failures

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


Frequently Asked Questions

How long does it take to guide developers?

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

What are the most common mistakes to avoid?

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

Do I need prior experience to follow this guide?

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

Is this approach secure enough for production?

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

Where can I get help if I run into issues?

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

Related Articles

- [Tor Hidden Services: How to Access Safely](/tor-hidden-services-how-to-access-safely/)
- [Best Browser To Use With Tor Hidden Services](/best-browser-to-use-with-tor-hidden-services/)
- [How To Set Up Onion Routing For Email Using Tor Hidden](/how-to-set-up-onion-routing-for-email-using-tor-hidden-servi/)
- [Onionshare Secure File Sharing Over Tor Network Setup](/onionshare-secure-file-sharing-over-tor-network-setup-and-us/)
- [Best Browser for Tor Network 2026: A Technical Guide](/best-browser-for-tor-network-2026/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
