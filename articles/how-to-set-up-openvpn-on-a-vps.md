---
layout: default
title: "How to Set Up OpenVPN on a VPS"
description: "Deploy a personal OpenVPN server on a VPS with certificate-based authentication, TLS hardening, and split tunneling configuration"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-openvpn-on-a-vps/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, vpn]
---

{% raw %}

Running your own OpenVPN server on a VPS gives you a private VPN that no commercial provider can log. This guide uses Easy-RSA for certificate management, TLS authentication to block port scans, and modern cipher selection.

## Prerequisites

- Ubuntu 22.04 VPS with root access
- A public IPv4 address
- UDP port 1194 open
- Clients with OpenVPN client software

## Step 1: Install OpenVPN and Easy-RSA

```bash
sudo apt update && sudo apt install openvpn easy-rsa
openvpn --version | head -1
```

## Step 2: Set Up the Certificate Authority

```bash
mkdir -p ~/openvpn-ca
cp -r /usr/share/easy-rsa/* ~/openvpn-ca/
cd ~/openvpn-ca

./easyrsa init-pki
./easyrsa build-ca nopass
# Enter "OpenVPN-CA" as Common Name
```

## Step 3: Generate Server Certificate and Keys

```bash
cd ~/openvpn-ca

./easyrsa gen-req server nopass
./easyrsa sign-req server server

# DH parameters
./easyrsa gen-dh

# TLS auth key
openvpn --genkey secret ta.key

# Copy to OpenVPN directory
sudo cp pki/ca.crt pki/issued/server.crt pki/private/server.key \
  pki/dh.pem ta.key /etc/openvpn/server/
sudo chmod 600 /etc/openvpn/server/*.key /etc/openvpn/server/*.pem
```

## Step 4: Generate Client Certificate

```bash
cd ~/openvpn-ca
./easyrsa gen-req alice nopass
./easyrsa sign-req client alice
# Client needs: ca.crt, alice.crt, alice.key, ta.key
```

## Step 5: Server Configuration

Create `/etc/openvpn/server/server.conf`:

```conf
port 1194
proto udp
dev tun

ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/server.crt
key /etc/openvpn/server/server.key
dh /etc/openvpn/server/dh.pem

tls-auth /etc/openvpn/server/ta.key 0
tls-version-min 1.2
tls-cipher TLS-ECDHE-ECDSA-WITH-AES-256-GCM-SHA384:TLS-ECDHE-RSA-WITH-AES-256-GCM-SHA384

cipher AES-256-GCM
ncp-ciphers AES-256-GCM:AES-128-GCM

server 10.8.0.0 255.255.255.0

push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 1.1.1.1"
push "dhcp-option DNS 1.0.0.1"

keepalive 10 120

user nobody
group nogroup
persist-key
persist-tun

status /var/log/openvpn/status.log 30
log-append /var/log/openvpn/openvpn.log
verb 3

# Compression disabled (VORACLE vulnerability)
compress
```

## Step 6: Enable IP Forwarding and NAT

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Get your public interface name
ip route get 8.8.8.8 | grep dev | awk '{print $5}'

# Add NAT rule (replace eth0 with your interface)
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

sudo apt install iptables-persistent
sudo netfilter-persistent save
```

## Step 7: Start OpenVPN

```bash
sudo mkdir -p /var/log/openvpn
sudo systemctl enable --now openvpn-server@server
sudo systemctl status openvpn-server@server
sudo journalctl -u openvpn-server@server -f
```

## Step 8: Create Client Configuration File

```bash
#!/bin/bash
# generate-client.sh
CLIENT="alice"
CA_DIR="$HOME/openvpn-ca"
SERVER_IP="YOUR_VPS_IP"

cat > ${CLIENT}.ovpn <<EOF
client
dev tun
proto udp
remote ${SERVER_IP} 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
tls-version-min 1.2
cipher AES-256-GCM
verb 3

<ca>
$(cat ${CA_DIR}/pki/ca.crt)
</ca>

<cert>
$(cat ${CA_DIR}/pki/issued/${CLIENT}.crt)
</cert>

<key>
$(cat ${CA_DIR}/pki/private/${CLIENT}.key)
</key>

<tls-auth>
$(cat ${CA_DIR}/ta.key)
</tls-auth>
key-direction 1
EOF

echo "Client config written to ${CLIENT}.ovpn"
```

## Step 9: Revoking a Client

```bash
cd ~/openvpn-ca
./easyrsa revoke alice
./easyrsa gen-crl

sudo cp pki/crl.pem /etc/openvpn/server/
# Add to server.conf: crl-verify /etc/openvpn/server/crl.pem
sudo systemctl restart openvpn-server@server
```

## Verify

```bash
ip addr show tun0
sudo cat /var/log/openvpn/status.log
# From client: curl https://ifconfig.me (should show VPS IP)
```

## Split Tunneling: Route Only Specific Traffic Through VPN

Full tunnel (redirect-gateway) routes all client traffic through the VPS. Split tunneling sends only specific subnets through the VPN, leaving other traffic on the client's local internet.

Remove the redirect-gateway push directive and replace it with specific routes:

```conf
# server.conf — split tunnel example
# Do NOT push redirect-gateway

# Route internal subnet through VPN
push "route 10.0.0.0 255.255.0.0"
push "route 192.168.1.0 255.255.255.0"

# Push DNS only for internal resolution
push "dhcp-option DNS 10.8.0.1"
```

On the client, the `.ovpn` file should NOT include `redirect-gateway`. Traffic to `10.0.0.0/16` goes through the VPN; everything else hits the client's ISP directly.

Use split tunneling when:
- Clients need access to internal services (home lab, office network) but want normal internet speed for other traffic
- You want to minimize VPS bandwidth consumption
- Full tunnel creates latency for services geographically close to the client

Verify routing after connection:

```bash
# On client after connecting
ip route show
# Should see: 10.0.0.0/16 via 10.8.0.1 dev tun0
# Default route should still point to local gateway, not tun0
curl https://ifconfig.me  # Should show client's ISP IP, not VPS IP
```

## Hardening: Block DNS Leaks and IPv6

A DNS leak occurs when the client resolves DNS through the ISP's resolver even while connected to VPN. Push DNS servers through the tunnel and disable IPv6 to prevent leaks:

```conf
# server.conf additions
push "dhcp-option DNS 10.8.0.1"
push "dhcp-option DNS 1.1.1.1"
push "block-outside-dns"  # Windows only, prevents DNS leaks on Windows clients

# Disable IPv6 pushing to prevent v6 leaks
tun-ipv6 no
```

On the VPS, run an unbound or dnsmasq resolver that listens on `10.8.0.1` to handle DNS inside the tunnel:

```bash
sudo apt install unbound
# /etc/unbound/unbound.conf.d/openvpn.conf
cat <<'EOF' | sudo tee /etc/unbound/unbound.conf.d/openvpn.conf
server:
    interface: 10.8.0.1
    access-control: 10.8.0.0/24 allow
    access-control: 127.0.0.1/8 allow
    do-ip4: yes
    do-ip6: no
    hide-identity: yes
    hide-version: yes
EOF
sudo systemctl enable --now unbound
```

Clients verify there are no leaks with:

```bash
# Test on client after connecting
dig +short whoami.akamai.net  # Should resolve through VPS IP
# Or use https://dnsleaktest.com
```

## Related Articles

- [Configure Openvpn With Obfuscation For Censored Networks](/how-to-configure-openvpn-with-obfuscation-for-censored-networks/)
- [OpenVPN Access Server vs Community Edition](/openvpn-access-server-vs-community-edition-differences-features-2026/)
- [Openvpn Push Route Configuration Selective Routing Explained](/openvpn-push-route-configuration-selective-routing-explained-step-by-step/)
- [Set Up a Personal VPN with WireGuard](/wireguard-personal-vpn-setup-guide)
- [Openvpn Tls Auth Vs Tls Crypt Difference Security Comparison](/openvpn-tls-auth-vs-tls-crypt-difference-security-comparison/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
