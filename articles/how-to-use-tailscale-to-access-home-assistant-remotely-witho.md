---
layout: default
title: "How to Use Tailscale to Access Home Assistant Remotely."
description: "A practical guide for developers and power users on setting up secure remote access to Home Assistant using Tailscale mesh VPN technology."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-use-tailscale-to-access-home-assistant-remotely-witho/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
---

{% raw %}
Running Home Assistant locally provides excellent privacy and control, but accessing it remotely traditionally requires opening ports on your router—a security risk that exposes your network to potential attacks. Tailscale offers a elegant solution by creating a secure mesh VPN that lets you access your Home Assistant instance from anywhere without port forwarding.

This guide walks through setting up Tailscale for remote Home Assistant access, targeting developers and power users who want enterprise-grade security without complexity.

## Why Avoid Port Forwarding

Opening ports on your router directly exposes services to the internet. Even with strong passwords, port scans and automated attacks target known vulnerabilities. Home Assistant running on your local network should remain private—Tailscale creates an encrypted tunnel that keeps your smart home infrastructure invisible to the public internet while remaining accessible to authorized devices.

## Understanding the Tailscale Architecture

Tailscale operates as a mesh VPN, creating direct encrypted connections between devices. Unlike traditional VPNs that route all traffic through a central server, Tailscale establishes peer-to-peer connections when possible, minimizing latency. Each device gets a unique Tailscale IP address (in the 100.x.x.x range), and your Home Assistant instance becomes reachable at that private IP from any device logged into your Tailscale network.

## Installing Tailscale on Home Assistant

The easiest method uses the Home Assistant Operating System with the Tailscale add-on. Access your Home Assistant instance, navigate to Settings → Add-ons, and search for "Tailscale." Install the official add-on and configure it with your Tailscale authentication key.

Configuration requires minimal settings:

```yaml
 tailscale:
   api_key: your_authentication_key
   accept_dns: true
   accept_routes: true
   verbose: false
```

After starting the add-on, note the Tailscale IP address assigned to your Home Assistant instance—you'll use this for connections.

## Alternative: Tailscale on a Raspberry Pi or Docker

For Home Assistant installations running on generic Linux, install Tailscale directly. On Debian/Ubuntu systems:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --accept-routes
```

Docker containers work similarly. Use the official Tailscale Docker image:

```bash
docker run -d \
  --name tailscale \
  --cap-add NET_ADMIN \
  --device /dev/net/tun \
  -e TS_AUTH_ONCE=true \
  -v /var/lib/tailscale:/var/lib/tailscale \
  -v /dev/net/tun:/dev/net/tun \
  tailscale/tailscale:latest \
  tailscale up --accept-routes
```

## Setting Up Client Devices

Install Tailscale on devices you want to use for remote access—phones, laptops, tablets. The Tailscale client is available for macOS, Windows, Linux, iOS, and Android. Log in using the same Tailscale account that authenticated your Home Assistant device.

Your Tailscale network now contains your Home Assistant instance and all your client devices. Each device receives a private IP address visible only within your Tailscale network.

## Accessing Home Assistant

With Tailscale running on both your Home Assistant server and client device, accessing your smart home is straightforward. Instead of your public IP address or domain name, use the Tailscale IP assigned to your Home Assistant instance.

The connection uses the format `http://[tailscale-ip]:8123`. For example, if your Home Assistant Tailscale IP is `100.64.123.456`, access it at `http://100.64.123.456:8123` from any device on your Tailscale network.

## Configuring Home Assistant for Tailscale

For optimal experience, configure Home Assistant to recognize Tailscale connections properly. Update your `configuration.yaml` to ensure proper hostname handling:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 100.64.0.0/10
```

This allows Home Assistant to correctly identify client devices behind the Tailscale VPN, enabling proper IP-based automation triggers and logging.

## Using Tailscale DNS for Convenience

Tailscale includes a DNS feature that assigns hostnames to devices. By default, your Home Assistant instance becomes accessible at `homeassistant.tail-scale.tset.hot` (your specific subdomain appears in the Tailscale admin console). Access Home Assistant at `http://homeassistant.tail-scale.tset.hot:8123`—much easier than memorizing IP addresses.

Enable this in your Tailscale admin panel or by using `--accept-dns` when running `tailscale up`.

## Security Considerations

Tailscale provides several security layers worth understanding. All traffic between devices is encrypted using WireGuard protocol, providing modern cryptographic protection. Access control happens through your Tailscale account—only users you authorize can connect to your network.

Consider enabling two-factor authentication on your Tailscale account for additional protection. The key expiration feature allows you to set time limits on authentication keys, useful for temporary access scenarios.

Review the devices connected to your Tailscale network regularly through the admin console. Remove devices you no longer use or recognize.

## Troubleshooting Common Issues

Connection problems typically stem from a few common causes. Verify both devices show as "Online" in the Tailscale admin console. Check that `accept_routes` is enabled on the Home Assistant Tailscale installation—this allows client devices to reach the Home Assistant network segment.

If peer-to-peer connection fails, Tailscale automatically falls back to relay servers (DERP), which may increase latency. This usually indicates firewall issues on one end. For best performance, ensure UDP ports 41641 are open on your network.

## Advanced: Tailscale SSH and Emergency Access

Beyond web access, Tailscale enables secure SSH into your Home Assistant server. Configure `tailscale ssh` to access your device's command line from anywhere—no need for traditional SSH keys or exposed SSH ports.

For scenarios requiring access without the Tailscale client, consider the Tailscale web proxy feature, which generates temporary URLs accessible through any browser—useful for one-off access on devices where installing Tailscale isn't practical.

## Conclusion

Tailscale transforms how you access Home Assistant remotely. Rather than exposing your network to internet scans and attacks, you get encrypted, authenticated access that works seamlessly across devices. The peer-to-peer architecture ensures good performance, and the zero-config setup means minimal maintenance once configured.

Your smart home remains private, your access remains secure, and you gain the ability to manage your Home Assistant instance from anywhere using the same credentials across all your devices.
{% endraw %}


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
