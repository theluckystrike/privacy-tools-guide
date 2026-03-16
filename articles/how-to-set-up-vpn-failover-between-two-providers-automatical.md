---
layout: default
title: "How to Set Up VPN Failover Between Two Providers."
description: "A practical guide for developers and power users to configure automatic VPN failover using wireguard, systemd, and custom scripts."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-vpn-failover-between-two-providers-automatical/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Set up automatic VPN failover by configuring two WireGuard providers with a health check script and systemd service that monitors connectivity and switches to backup within seconds of primary failure. This eliminates manual intervention and ensures uninterrupted VPN protection.

## Understanding VPN Failover Architecture

VPN failover works by monitoring the active connection and automatically switching to a backup when the primary fails. The key components are:

- **Primary VPN tunnel**: Your main connection (typically WireGuard or OpenVPN)
- **Backup VPN tunnel**: Secondary provider as a standby
- **Health check script**: Monitors connectivity and triggers failover
- **Routing policy**: Ensures traffic uses the active tunnel

## Prerequisites

Ensure you have:

- Two VPN provider accounts (any WireGuard-compatible provider works)
- A Linux system with WireGuard tools installed
- Root or sudo access
- Basic familiarity with the command line

Install WireGuard tools on Ubuntu/Debian:

```bash
sudo apt update
sudo apt install wireguard-tools curl
```

## Step 1: Configure Primary VPN

Generate your WireGuard configuration for the primary provider. Most providers give you a config file. Save it as `/etc/wireguard/wg0.conf`:

```bash
sudo nano /etc/wireguard/wg0.conf
```

Add your provider's configuration:

```ini
[Interface]
PrivateKey = YOUR_PRIMARY_PRIVATE_KEY
Address = 10.0.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = PROVIDER_PUBLIC_KEY
Endpoint = vpn.provider1.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Start the primary interface:

```bash
sudo wg-quick up wg0
sudo wg show
```

## Step 2: Configure Backup VPN

Create a second WireGuard configuration for your backup provider at `/etc/wireguard/wg1.conf`:

```bash
sudo nano /etc/wireguard/wg1.conf
```

```ini
[Interface]
PrivateKey = YOUR_BACKUP_PRIVATE_KEY
Address = 10.0.1.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = PROVIDER_PUBLIC_KEY
Endpoint = vpn.provider2.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

## Step 3: Create Health Check Script

The health check script monitors connectivity and decides when to switch. Create `/usr/local/bin/vpn-failover.sh`:

```bash
#!/bin/bash

PRIMARY_INTERFACE="wg0"
BACKUP_INTERFACE="wg1"
CHECK_HOST="1.1.1.1"
MAX_FAILURES=3

check_connectivity() {
    local interface=$1
    ping -I "$interface" -c 2 -W 3 "$CHECK_HOST" > /dev/null 2>&1
    return $?
}

get_active_interface() {
    ip route show default | awk '/default/ {print $5}' | head -1
}

failover() {
    echo "$(date): Failover triggered - switching to $BACKUP_INTERFACE"
    sudo wg-quick down "$PRIMARY_INTERFACE"
    sudo wg-quick up "$BACKUP_INTERFACE"
    echo "$(date): Switched to $BACKUP_INTERFACE" >> /var/log/vpn-failover.log
}

failback() {
    echo "$(date): Failback triggered - switching to $PRIMARY_INTERFACE"
    sudo wg-quick down "$BACKUP_INTERFACE"
    sleep 2
    sudo wg-quick up "$PRIMARY_INTERFACE"
    echo "$(date): Switched to $PRIMARY_INTERFACE" >> /var/log/vpn-failover.log
}

main() {
    active=$(get_active_interface)
    failures=0

    while true; do
        if [[ "$active" == "$PRIMARY_INTERFACE" ]]; then
            if ! check_connectivity "$PRIMARY_INTERFACE"; then
                failures=$((failures + 1))
                echo "$(date): Primary check failed ($failures/$MAX_FAILURES)"
                if [[ $failures -ge $MAX_FAILURES ]]; then
                    failover
                    active="$BACKUP_INTERFACE"
                    failures=0
                fi
            else
                failures=0
            fi
        elif [[ "$active" == "$BACKUP_INTERFACE" ]]; then
            if check_connectivity "$PRIMARY_INTERFACE"; then
                echo "$(date): Primary recovered, failing back"
                failback
                active="$PRIMARY_INTERFACE"
            fi
        fi
        sleep 10
    done
}

main
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/vpn-failover.sh
```

## Step 4: Create Systemd Service

Create a systemd service to run the failover script as a daemon:

```bash
sudo nano /etc/systemd/system/vpn-failover.service
```

```ini
[Unit]
Description=VPN Failover Monitor
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/vpn-failover.sh
Restart=always
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable vpn-failover.service
sudo systemctl start vpn-failover.service
```

Check the status:

```bash
sudo systemctl status vpn-failover.service
```

## Step 5: Verify Configuration

Test your setup by intentionally blocking the primary connection:

```bash
# Check which interface is active
ip route show default

# View failover logs
sudo tail -f /var/log/vpn-failover.log

# Manually trigger failover
sudo wg-quick down wg0
# The script should automatically bring up wg1
```

## Advanced: IPTable Rules for Guaranteed Routing

To ensure traffic always uses the active VPN regardless of interface ordering, add specific routing rules:

```bash
# Ensure all traffic uses the default route through active VPN
sudo ip rule add from all table main prio 100

# Or for specific use cases, mark packets
sudo iptables -t mangle -A PREROUTING -s 192.168.1.0/24 -j MARK --set-mark 1
sudo ip rule add fwmark 1 table vpn_table
```

## Troubleshooting Common Issues

**Problem**: Both VPNs connect but traffic still goes through the original interface.

**Solution**: Check your routing table with `ip route show table all` and ensure WireGuard is set as the default route.

**Problem**: Failover triggers too slowly.

**Solution**: Reduce the sleep interval in the script and decrease MAX_FAILURES to 2.

**Problem**: DNS leaks after failover.

**Solution**: Ensure your WireGuard configs include DNS settings and consider using a system-wide DNS resolver like dnsmasq.

## Performance Considerations

Failover introduces brief interruptions (typically 5-15 seconds). For applications requiring zero downtime, consider:

- **BGP peering**: More complex but faster failover
- **Anycast IPs**: Providers offering anycast reduce latency during switches
- **Connection pooling**: Applications should implement retry logic

## Security Notes

- Store private keys securely with proper file permissions: `chmod 600 /etc/wireguard/*.conf`
- Consider encrypting the failover log if it contains sensitive information
- Regularly rotate VPN credentials
- Audit your firewall rules to ensure only intended traffic exits the VPN

## Conclusion

Automatic VPN failover provides resilience for critical connections. The setup above uses WireGuard for its speed and simplicity, but the same principles apply to OpenVPN or other protocols. Monitor your logs regularly and adjust failure thresholds based on your network conditions.

For production environments, consider combining this with network monitoring tools like Prometheus or Zabbix to track VPN health metrics over time.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
