---
layout: default
title: "How to Use SSH Tunneling for Encrypted Communication Between Devices"
description: "A practical guide to SSH tunneling for developers and power users. Learn local, remote, and dynamic port forwarding with real-world examples."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-use-ssh-tunneling-for-encrypted-communication-between/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
intent-checked: true
---

{% raw %}

SSH tunneling provides a powerful method for creating encrypted communication channels between devices. By forwarding traffic through an SSH connection, you can secure data transmission over untrusted networks, access services on remote machines without exposing them to the internet, and bypass firewall restrictions. This guide covers the three main types of SSH tunnels with practical examples for developers and power users.

## Understanding SSH Tunneling Basics

SSH tunneling works by encapsulating unencrypted traffic inside an encrypted SSH connection. When you create a tunnel, SSH acts as a proxy that forwards traffic from a local port to a remote destination through the encrypted channel. This approach leverages SSH's built-in encryption, which uses strong algorithms like AES-256-GCM or ChaCha20-Poly1305.

The basic syntax for creating an SSH tunnel involves the `-L` (local), `-R` (remote), or `-D` (dynamic) flags with the `ssh` command. Each flag serves a different use case depending on where you need to access services and how you want to route traffic.

## Local Port Forwarding (-L Flag)

Local port forwarding allows you to access a remote service as if it were running locally. This is useful when you need to connect to a service on a remote server that isn't directly accessible from your current network.

### Basic Local Forwarding Syntax

```bash
ssh -L local_port:destination_host:destination_port user@ssh_server
```

This command forwards connections from your local machine's `local_port` to the `destination_host` on the remote network, through the SSH server.

### Practical Example: Accessing a Remote Database

Imagine you have a MySQL database running on a remote server at 192.168.1.100 that you need to access securely:

```bash
ssh -L 3306:192.168.1.100:3306 user@remote-server.com
```

After establishing this tunnel, you can connect to the database as if it were running locally:

```bash
mysql -h 127.0.0.1 -u database_user -p
```

The connection travels encrypted from your machine to `remote-server.com`, then to the database server. This approach is particularly valuable when working with production databases over public networks.

### Another Example: Accessing Internal Web Services

```bash
ssh -L 8080:internal-web-server:80 user@bastion-host.com
```

After running this command, visiting `http://127.0.0.1:8080` in your browser displays the website from `internal-web-server` behind the bastion host. Developers frequently use this pattern to access development environments, staging servers, or internal dashboards.

## Remote Port Forwarding (-R Flag)

Remote port forwarding does the opposite of local forwarding—it allows external users to access a service on your local machine through the SSH server. This is essential for exposing local development servers to the internet or sharing services with colleagues without configuring router port forwarding.

### Basic Remote Forwarding Syntax

```bash
ssh -R remote_port:local_host:local_port user@ssh_server
```

This makes your local service available at `remote_port` on the SSH server.

### Practical Example: Sharing a Local Web Server

Suppose you're developing a web application locally on port 3000 and want a colleague to test it:

```bash
ssh -R 8080:localhost:3000 user@public-server.com
```

Your colleague can now access your local application by visiting `http://public-server.com:8080`. The traffic routes through the SSH server to your machine, remaining encrypted throughout.

### Exposing a Service to a Specific Remote Port

```bash
ssh -R 9000:localhost:22 user@remote-server.com
```

This makes your local SSH port accessible via port 9000 on the remote server, enabling you to connect back to your local machine from the remote server.

## Dynamic Port Forwarding (-D Flag)

Dynamic port forwarding turns your SSH server into a SOCKS proxy server, allowing you to route traffic through the SSH connection for any application that supports SOCKS proxies. This is particularly useful for secure browsing, accessing services behind firewalls, and encrypting all application traffic.

### Basic Dynamic Forwarding Syntax

```bash
ssh -D local_socks_port user@ssh_server
```

This creates a SOCKS proxy listening on `local_socks_port`.

### Practical Example: Creating an Encrypted SOCKS Proxy

```bash
ssh -D 1080 user@example-server.com
```

After establishing this tunnel, configure your applications to use `127.0.0.1:1080` as a SOCKS5 proxy. All traffic through this proxy travels encrypted through the SSH connection.

### Configuring Browsers to Use the SOCKS Proxy

For Firefox:

1. Go to Settings → Network Settings
2. Select "Manual proxy configuration"
3. Set SOCKS Host to `127.0.0.1` and Port to `1080`
4. Choose "SOCKS v5"

For curl command-line usage:

```bash
curl --socks5 127.0.0.1:1080 https://example.com
```

This routes your HTTP/HTTPS requests through the encrypted tunnel, protecting your browsing from eavesdroppers on untrusted networks.

## Persisting SSH Tunnels

SSH tunnels close when the SSH session ends. For persistent tunnels, use one of these approaches:

### Using AutoSSH

AutoSSH automatically restarts tunnels if they fail:

```bash
autossh -M 20000 -f -N -L 3306:192.168.1.100:3306 user@remote-server.com
```

The `-M 20000` flag specifies a monitoring port. Install autossh with `apt install autossh` on Debian/Ubuntu or `brew install autossh` on macOS.

### Using SSH Config Files

For frequently used tunnels, add them to your SSH config (`~/.ssh/config`):

```
Host db-tunnel
    HostName remote-server.com
    User user
    LocalForward 3306 192.168.1.100:3306
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

Now connect using `ssh db-tunnel` and the tunnel establishes automatically.

### Running Tunnels in the Background

Use `-f` to background the SSH process:

```bash
ssh -f -N -L 8080:internal-server:80 user@bastion-host.com
```

Combine this with system startup scripts or systemd units for automatic tunnel establishment on boot.

## Security Considerations

When using SSH tunnels, keep these practices in mind:

- **Use key-based authentication**: Disable password authentication and use SSH keys for stronger security
- **Limit bind addresses**: Use `127.0.0.1` instead of `0.0.0.0` to prevent unauthorized access
- **Configure idle timeouts**: Set `ServerAliveInterval` and `ClientAliveInterval` to detect dropped connections
- **Restrict tunnel permissions**: Use `permitopen` in `~/.ssh/authorized_keys` to restrict which ports users can forward

Example restricted authorized_keys entry:

```
command="echo 'Port forwarding restricted'",permitopen="127.0.0.1:3306" ssh-rsa AAAA...
```

This limits the user to forwarding only port 3306 on localhost.

## Common Use Cases for Developers

SSH tunneling addresses several practical scenarios:

1. **Database administration**: Securely connect to production databases from your local machine
2. **Development workflows**: Access internal development servers without VPN configuration
3. **Remote collaboration**: Share local development environments with team members
4. **Testing webhooks**: Expose local webhook endpoints for third-party service testing
5. **Legacy system access**: Connect to systems that don't support modern encryption protocols

## Troubleshooting Tunnel Connections

When tunnels don't work as expected, verify these common issues:

- Check that the SSH server allows port forwarding (`AllowTcpForwarding yes` in sshd_config)
- Confirm firewall rules permit traffic on the involved ports
- Ensure the destination service is actually running on the remote host
- Test connectivity with `telnet destination_host port` or `nc -zv destination_host port`

For debugging, add verbose output:

```bash
ssh -v -L 8080:internal-server:80 user@bastion-host.com
```

The `-v` flag increases verbosity and helps identify connection issues.

---

SSH tunneling remains one of the most reliable methods for creating encrypted communication channels between devices. Whether you need simple local forwarding for database access or dynamic SOCKS proxies for secure browsing, SSH provides the encryption and flexibility required for modern development workflows.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
