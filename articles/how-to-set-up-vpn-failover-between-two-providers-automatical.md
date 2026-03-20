---

layout: default
title: "How To Set Up Vpn Failover Between Two Providers Automatical"
description: "A practical guide for developers and power users to implement automatic VPN failover between two providers using open-source tools and scripts."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-vpn-failover-between-two-providers-automatical/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Set up automatic VPN failover by configuring two WireGuard tunnels (wg0 and wg1) on a Linux machine, then running a bash monitoring script as a systemd service that pings through the active tunnel every 10 seconds and switches the default route to the backup provider on failure. This eliminates manual intervention and restores connectivity within seconds of a provider outage. The full implementation uses open-source tools and works with most VPN providers.

## Why Automatic Failover Matters

VPN connections serve as critical infrastructure for remote access, site-to-site networking, and privacy. Network interruptions happen—provider outages, ISP issues, or hardware failures can disconnect your tunnel. Manual switching wastes time and introduces human error. An automatic failover system detects connectivity loss within seconds and restores your connection through an alternate provider.

The solution described here uses **WireGuard** for its simplicity and performance, combined with a monitoring script that tracks tunnel health. You can adapt these principles to OpenVPN or other protocols as needed.

## Prerequisites

Before implementing failover, gather the following:

- Two active VPN provider accounts (or self-hosted VPN servers)
- A Linux machine to run the failover script (a VPS, Raspberry Pi, or dedicated hardware)
- Basic familiarity with terminal commands and cron scheduling
- Root or sudo access on your Linux system

## Setting Up Your Primary and Secondary VPN Tunnels

First, establish both VPN connections on your Linux machine. This example uses WireGuard, but the principles apply to any protocol.

### Install WireGuard

```bash
# Debian/Ubuntu
sudo apt update
sudo apt install wireguard

#RHEL/Fedora
sudo dnf install wireguard-tools
```

### Configure the Primary Tunnel

Create the WireGuard configuration for your primary provider:

```bash
sudo nano /etc/wireguard/wg0.conf
```

Add your provider's settings:

```ini
[Interface]
PrivateKey = YOUR_PRIMARY_PRIVATE_KEY
Address = 10.0.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = PRIMARY_SERVER_PUBLIC_KEY
Endpoint = primary.vpn-provider.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

### Configure the Secondary Tunnel

Create a second WireGuard interface for your backup provider:

```bash
sudo nano /etc/wireguard/wg1.conf
```

Add the secondary provider configuration:

```ini
[Interface]
PrivateKey = YOUR_SECONDARY_PRIVATE_KEY
Address = 10.0.1.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = SECONDARY_SERVER_PUBLIC_KEY
Endpoint = secondary.vpn-provider.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Bring up both interfaces:

```bash
sudo wg-quick up wg0
sudo wg-quick up wg1
```

## Implementing the Failover Script

The core of your failover system is a monitoring script that checks connectivity through each tunnel and switches routes when needed.

Create the failover script:

```bash
sudo nano /usr/local/bin/vpn-failover.sh
```

Add this content:

```bash
#!/bin/bash

# Configuration
PRIMARY_IF="wg0"
SECONDARY_IF="wg1"
PRIMARY_GW="10.0.0.1"
CHECK_HOST="8.8.8.8"
CHECK_INTERVAL=10
LOG_FILE="/var/log/vpn-failover.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

check_connectivity() {
    local interface=$1
    local gateway=$2
    ip route get "$CHECK_HOST" | grep -q "$interface" && ping -I "$interface" -c 3 -W 5 "$CHECK_HOST" > /dev/null 2>&1
    return $?
}

get_active_tunnel() {
    ip route show default | grep -oP '(wg[01])' | head -1
}

failover() {
    local from_if=$1
    local to_if=$2
    
    log "FAILOVER: $from_if failed, switching to $to_if"
    
    # Remove default routes through failed tunnel
    sudo ip route del default dev "$from_if" 2>/dev/null
    
    # Add default route through backup tunnel
    sudo ip route add default dev "$to_if"
    
    # Restart the failed interface for recovery
    sudo wg-quick down "$from_if" 2>/dev/null
    sleep 2
    sudo wg-quick up "$from_if" 2>/dev/null
}

# Main monitoring loop
while true; do
    ACTIVE=$(get_active_tunnel)
    
    if [ "$ACTIVE" = "$PRIMARY_IF" ]; then
        if ! check_connectivity "$PRIMARY_IF" "$PRIMARY_GW"; then
            log "Primary tunnel ($PRIMARY_IF) is down"
            failover "$PRIMARY_IF" "$SECONDARY_IF"
        fi
    elif [ "$ACTIVE" = "$SECONDARY_IF" ]; then
        if ! check_connectivity "$SECONDARY_IF" "$PRIMARY_GW"; then
            log "Secondary tunnel ($SECONDARY_IF) is down"
            # Try to restore primary
            sudo ip route del default dev "$SECONDARY_IF"
            sudo ip route add default dev "$PRIMARY_IF"
        fi
    fi
    
    sleep "$CHECK_INTERVAL"
done
```

Make the script executable:

```bash
sudo chmod +x /usr/local/bin/vpn-failover.sh
```

## Running the Failover Monitor

Start the failover script as a systemd service for reliability. Create the service file:

```bash
sudo nano /etc/systemd/system/vpn-failover.service
```

Add:

```ini
[Unit]
Description=VPN Failover Monitor
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/vpn-failover.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable vpn-failover
sudo systemctl start vpn-failover
```

## Testing Your Failover System

Verify the failover mechanism works before relying on it in production.

### Simulate a Failure

Disconnect your primary VPN to trigger failover:

```bash
sudo wg-quick down wg0
```

Check the logs:

```bash
sudo journalctl -u vpn-failover -f
```

You should see the failover script detect the failure and switch to the secondary tunnel. Confirm your public IP changed by running:

```bash
curl ifconfig.me
```

### Restore Primary Connectivity

Bring the primary tunnel back up:

```bash
sudo wg-quick up wg0
```

The script will detect recovery and switch back, or you can configure it to remain on the secondary tunnel until manually restored.

## Additional Considerations

**Routing Policy**: Modify the script to prioritize specific traffic through either tunnel based on destination, source IP, or application requirements using `ip rule` tables.

**Health Check Frequency**: Adjust the `CHECK_INTERVAL` based on your tolerance for downtime. A 10-second interval balances responsiveness with system overhead.

**Multiple Failures**: Extend the script to support additional VPN providers by adding more interfaces and corresponding checks.

**Logging**: Monitor the log file regularly or integrate with a log aggregation system for production environments.

## Policy Routing for Split Traffic

Use `ip rule` with named routing tables to send specific traffic through each tunnel independently of the default route:

```bash
# Register named routing tables
echo "200 vpn0" | sudo tee -a /etc/iproute2/rt_tables
echo "201 vpn1" | sudo tee -a /etc/iproute2/rt_tables

# Route traffic from wg0 IP through provider 1 table
sudo ip rule add from 10.0.0.2 table vpn0
sudo ip route add default dev wg0 table vpn0

# Route traffic from wg1 IP through provider 2 table
sudo ip rule add from 10.0.1.2 table vpn1
sudo ip route add default dev wg1 table vpn1

# Verify active rules
ip rule show
ip route show table vpn0
```

This routes sensitive API traffic through provider 1 and general traffic through provider 2, without modifying the global default route. Add rules to `/etc/rc.local` or a systemd oneshot service for persistence across reboots.

Automatic failover with WireGuard and systemd eliminates single-point failures. Swap providers by updating the `.conf` files — the monitoring script and routing tables stay unchanged.

## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
