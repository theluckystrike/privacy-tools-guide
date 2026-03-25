---
layout: default
title: "How to Set Up Nebula Mesh VPN for Teams"
description: "Deploy Nebula mesh VPN across your team with a lighthouse server, signed node certificates, and encrypted peer-to-peer tunnels"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-nebula-mesh-vpn-for-teams/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, vpn]
---

{% raw %}

Nebula is an open-source overlay network built by Slack's infrastructure team. It creates encrypted mesh tunnels directly between nodes using UDP hole-punching, with a central lighthouse that helps peers find each other without routing all traffic through it. This makes it fast, private, and suitable for distributed teams.

What You Need

- One VPS as the lighthouse (a small $5/month instance works)
- Linux or macOS on each team member's machine
- Root or sudo access on all nodes
- Ports: UDP 4242 open on the lighthouse

Step 1 - Install Nebula

Download the latest release on all nodes. On the lighthouse and each team machine:

```bash
On x86_64 Linux
wget https://github.com/slackhq/nebula/releases/latest/download/nebula-linux-amd64.tar.gz
tar -xzf nebula-linux-amd64.tar.gz
sudo mv nebula nebula-cert /usr/local/bin/
sudo chmod +x /usr/local/bin/nebula /usr/local/bin/nebula-cert
```

For macOS:
```bash
brew install nebula
```

Verify:
```bash
nebula-cert --version
```

Step 2 - Create the Certificate Authority

Run this once on a secure machine. not on the lighthouse. The CA key should never leave this machine.

```bash
mkdir -p ~/nebula-ca && cd ~/nebula-ca

Create the CA certificate (valid 5 years)
nebula-cert ca -name "MyTeam CA" -duration 43800h

ls -la
ca.crt  ca.key
```

Store `ca.key` in an encrypted location (e.g., a KeePassXC database or an offline machine). You only need it when creating new node certificates.

Step 3 - Sign Node Certificates

Each node needs a certificate with a unique Nebula IP. Use the `10.42.0.0/16` range (or any RFC1918 range you prefer).

```bash
Lighthouse
nebula-cert sign -name "lighthouse" -ip "10.42.0.1/24" -ca-crt ca.crt -ca-key ca.key

Team member Alice
nebula-cert sign -name "alice" -ip "10.42.0.2/24" -ca-crt ca.crt -ca-key ca.key

Team member Bob
nebula-cert sign -name "bob" -ip "10.42.0.3/24" -ca-crt ca.crt -ca-key ca.key

Server (e.g., internal wiki)
nebula-cert sign -name "wiki" -ip "10.42.0.10/24" \
  -groups "servers" -ca-crt ca.crt -ca-key ca.key
```

The `-groups` flag lets you write firewall rules targeting a set of nodes rather than individual IPs.

Each sign command produces `<name>.crt` and `<name>.key`. Distribute each pair securely to the matching machine along with `ca.crt`.

Step 4 - Configure the Lighthouse

Create `/etc/nebula/config.yml` on the lighthouse VPS:

```yaml
pki:
  ca: /etc/nebula/ca.crt
  cert: /etc/nebula/lighthouse.crt
  key: /etc/nebula/lighthouse.key

static_host_map:
  "10.42.0.1": ["YOUR_VPS_PUBLIC_IP:4242"]

lighthouse:
  am_lighthouse: true
  interval: 60

listen:
  host: 0.0.0.0
  port: 4242

punchy:
  punch: true
  respond: true
  delay: 1s

tun:
  disabled: false
  dev: nebula1
  drop_local_broadcast: false
  drop_multicast: false
  tx_queue: 500
  mtu: 1300

firewall:
  outbound:
    - port: any
      proto: any
      host: any
  inbound:
    - port: any
      proto: icmp
      host: any
    - port: any
      proto: any
      host: any
```

Copy the certificates into place:
```bash
sudo mkdir -p /etc/nebula
sudo cp ca.crt lighthouse.crt lighthouse.key /etc/nebula/
sudo chmod 600 /etc/nebula/lighthouse.key
```

Step 5 - Configure Team Member Nodes

Create `/etc/nebula/config.yml` on Alice's machine (and each other team member, adjusting cert names):

```yaml
pki:
  ca: /etc/nebula/ca.crt
  cert: /etc/nebula/alice.crt
  key: /etc/nebula/alice.key

static_host_map:
  "10.42.0.1": ["YOUR_VPS_PUBLIC_IP:4242"]

lighthouse:
  am_lighthouse: false
  interval: 60
  hosts:
    - "10.42.0.1"

listen:
  host: 0.0.0.0
  port: 4242

punchy:
  punch: true
  respond: true

tun:
  disabled: false
  dev: nebula1
  mtu: 1300

firewall:
  outbound:
    - port: any
      proto: any
      host: any
  inbound:
    - port: any
      proto: icmp
      host: any
    - port: 22
      proto: tcp
      host: any
    - port: 443
      proto: tcp
      groups:
        - servers
```

The inbound rules here allow ICMP from anyone on the mesh, SSH from any mesh peer, and HTTPS only from nodes in the `servers` group.

Step 6 - Start Nebula

On the lighthouse and all nodes:

```bash
Test first
sudo nebula -config /etc/nebula/config.yml -test

Run as a systemd service
sudo tee /etc/systemd/system/nebula.service > /dev/null <<EOF
[Unit]
Description=Nebula overlay network
After=network.target

[Service]
ExecStart=/usr/local/bin/nebula -config /etc/nebula/config.yml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now nebula
sudo systemctl status nebula
```

Step 7 - Verify Connectivity

From Alice's machine:
```bash
Ping lighthouse
ping -c 3 10.42.0.1

Ping Bob directly (should punch through NAT)
ping -c 3 10.42.0.2

Check Nebula interface
ip addr show nebula1

Check nebula logs for direct tunnel establishment
sudo journalctl -u nebula -f
```

Look for `Direct connection established` in the logs. this confirms peer-to-peer tunneling is working rather than relaying through the lighthouse.

Firewall Rules and Groups

Nebula's firewall is evaluated on the receiving node. You can restrict access by group membership:

```yaml
firewall:
  inbound:
    # Allow all team members to reach internal services
    - port: 8080
      proto: tcp
      groups:
        - team
    # Allow only servers to accept database connections
    - port: 5432
      proto: tcp
      groups:
        - servers
```

When you sign a certificate, add it to a group:
```bash
nebula-cert sign -name "carol" -ip "10.42.0.4/24" \
  -groups "team,developers" -ca-crt ca.crt -ca-key ca.key
```

Certificate Rotation

Certificates expire. To rotate before expiry:

```bash
Generate new cert for alice (keep same IP)
nebula-cert sign -name "alice" -ip "10.42.0.2/24" \
  -duration 8760h -ca-crt ca.crt -ca-key ca.key

Distribute alice.crt and alice.key to Alice's machine
Restart nebula on Alice's machine
sudo systemctl restart nebula
```

Revoking a Node

Nebula 1.x does not have a CRL mechanism. to revoke access, you must issue a new CA and re-sign all nodes. Plan your team structure so that you can re-roll certificates when a member departs.

One workaround - use short-lived certificates (e.g., 90-day) and automate renewal with a script that requires re-authentication before issuing a new cert.

Related Articles

- [VPN Kill Switch Configuration on Linux with iptables](/vpn-kill-switch-linux-iptables-setup/)
- [Use Mesh Networking for Private Communication](/how-to-use-mesh-networking-for-private-communication-without/)
- [How To Use Tcpdump To Verify Vpn Traffic Is Encrypted](/how-to-use-tcpdump-to-verify-vpn-traffic-is-encrypted/)
- [How To Set Up Vpn Failover Between Two Providers Automatical](/how-to-set-up-vpn-failover-between-two-providers-automatical/)
- [How to Configure VPN Exempt List for Local Network](/how-to-configure-vpn-exempt-list-for-local-network-access/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
