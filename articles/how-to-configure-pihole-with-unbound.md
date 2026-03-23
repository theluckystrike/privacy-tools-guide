---
layout: default
title: "How to Configure Pi-hole with Unbound DNS"
description: "Set up Pi-hole with a local Unbound recursive resolver to block ads and trackers while eliminating reliance on upstream DNS providers"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-configure-pihole-with-unbound-dns/
categories: [guides]
reviewed: true
score: 6
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

## DNS-Over-HTTPS Fallback with Unbound

For situations where your ISP or network filters port 53 UDP traffic, configure Unbound to use DNS-over-HTTPS (DoH) for its upstream lookups while still presenting a standard DNS interface to Pi-hole:

```bash
sudo apt install stubby

# Configure stubby as a DoH proxy on port 8053
sudo tee /etc/stubby/stubby.yml > /dev/null << 'EOF'
resolution_type: GETDNS_RESOLUTION_STUB
dns_transport_list:
  - GETDNS_TRANSPORT_HTTPS
tls_authentication: GETDNS_AUTHENTICATION_REQUIRED
tls_query_padding_blocksize: 128
listen_addresses:
  - 127.0.0.1@8053
round_robin_upstreams: 1
upstream_recursive_servers:
  - address_data: 1.1.1.1
    tls_port: 853
    tls_auth_name: "cloudflare-dns.com"
  - address_data: 9.9.9.9
    tls_port: 853
    tls_auth_name: "dns.quad9.net"
EOF

sudo systemctl enable --now stubby
```

Then configure Unbound to forward to stubby instead of root servers when DoH is needed:

```conf
# Add to /etc/unbound/unbound.conf.d/pi-hole.conf
forward-zone:
  name: "."
  forward-addr: 127.0.0.1@8053  # stubby's DoH proxy
  forward-no-cache: no
```

Note: Using a forwarder (stubby) means Unbound no longer does full recursive resolution. Disable this and use the root hints configuration whenever your network allows direct recursive DNS.

---

## Related Articles

- [Privacy-Focused DNS Resolver Comparison 2026](/privacy-dns-resolver-comparison-2026/)
- [Encrypted DNS over HTTPS on Linux](/encrypted-dns-over-https-linux-setup)
- [Home Network Privacy Pihole Dns Filtering Guide 2026](/home-network-privacy-pihole-dns-filtering-guide-2026/)
- [Configure Private DNS on Android for System-Wide Tracker](/how-to-configure-private-dns-on-android-for-system-wide-trac/)
- [Privacy-Focused DNS Providers Comparison 2026](/privacy-focused-dns-providers-comparison-2026/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
