---
layout: default
title: "Secure Shell Hardening Beyond SSH Config"
description: "Harden Linux shell access with PAM restrictions, login banners, session timeouts, sudo auditing, fail2ban tuning, and port-knocking for SSH exposure red..."
date: 2026-03-22
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /secure-shell-hardening-beyond-ssh-config/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Secure Shell Hardening Beyond SSH Config

SSH configuration (`/etc/ssh/sshd_config`) is the first step. key-only auth, no root login, specific ciphers. But the session that follows SSH authentication has its own attack surface. This guide covers what happens after the connection is established: PAM controls, sudo hardening, session logging, and reducing SSH exposure entirely.

PAM - Restrict Who Can Log In

PAM (Pluggable Authentication Modules) controls authentication and session setup.

```bash
/etc/security/access.conf. allow only specific users/groups
Format - permission : user/group : origin
Deny all except members of the sshusers group

/etc/pam.d/sshd. add this line near the top:
account    required     pam_access.so
```

```bash
/etc/security/access.conf
Allow members of sshusers from anywhere
+ : (sshusers) : ALL

Allow specific user from specific IP range
+ : alice : 10.0.0.0/24

Deny everyone else
- : ALL : ALL
```

```bash
Create the sshusers group and add users
sudo groupadd sshusers
sudo usermod -aG sshusers alice
sudo usermod -aG sshusers bob

Test PAM access (does not actually log in)
sudo pamtester login alice authenticate
```

Time-Based Login Restrictions

Limit SSH access to business hours using PAM time:

```bash
/etc/security/time.conf
Format - service;tty;users;time
Deny SSH for contractor account on weekends

sshd;*;contractor;!Sa-Su0000-2400
sshd;*;contractor;Wk0800-1800   # weekdays 8am-6pm only
```

```bash
/etc/pam.d/sshd. add:
account    required     pam_time.so
```

Session Timeouts and Idle Lockout

```bash
/etc/profile.d/autologout.sh. set for all users
readonly TMOUT=600   # 10-minute idle timeout, disconnect
export TMOUT

For SSH specifically (belt-and-suspenders with sshd_config):
ClientAliveInterval 300
ClientAliveCountMax 2
These are in sshd_config, not profile. but document both
```

```bash
Lock screen on inactivity using vlock (for console sessions)
sudo apt install vlock
Users can run - vlock -a  to lock all virtual consoles

For remote sessions, TMOUT in /etc/profile.d/ is the primary control
```

sudo Hardening

```bash
/etc/sudoers.d/hardened. do not edit /etc/sudoers directly
Always - sudo visudo -f /etc/sudoers.d/hardened

Require password every time (disable sudo timestamp caching)
Defaults        timestamp_timeout=0

Log all sudo commands (default) and log to both syslog and a file
Defaults        logfile="/var/log/sudo.log"
Defaults        log_input, log_output
Defaults        iolog_dir="/var/log/sudo-io/%{user}"

Require full path for commands
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

Prevent sudo from running interactive shells (su escalation)
Defaults        !visiblepw
Defaults        always_set_home
Defaults        env_reset

Per-user - allow specific commands only, no shell
alice   ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl reload nginx
bob     ALL=(ALL) PASSWD: /usr/bin/apt-get update, /usr/bin/apt-get upgrade
```

```bash
Test sudo policy
sudo -l -U alice   # list what alice can do
sudo -l            # list what you can do

Review sudo logs
sudo journalctl -u sudo
tail -f /var/log/sudo.log
```

Login Banner and Legal Warning

A proper login banner establishes that unauthorized access is prohibited. required for many compliance frameworks.

```bash
/etc/issue.net. displayed before SSH login
sudo tee /etc/issue.net << 'EOF'
*
AUTHORIZED USE ONLY

Unauthorized access to this system is prohibited. All activity is monitored
and logged. Disconnect immediately if you do not have authorization.

By continuing, you consent to monitoring in accordance with company policy.
*
EOF

Enable in sshd_config:
Banner /etc/issue.net

Local console banner (shown before login prompt)
/etc/issue. same content
sudo cp /etc/issue.net /etc/issue
```

SSH Session Logging with script

Every command run in a SSH session can be recorded:

```bash
/etc/profile.d/session-log.sh
Logs all sessions to /var/log/sessions/

LOGDIR="/var/log/sessions"
mkdir -p "$LOGDIR"

if [ -n "$SSH_CONNECTION" ] && [ -z "$SCRIPT" ]; then
    LOGFILE="${LOGDIR}/$(date +%Y%m%d-%H%M%S)-$(whoami)-$(echo $SSH_CONNECTION | awk '{print $1}').log"
    export SCRIPT="$LOGFILE"
    exec script -qf "$LOGFILE"
fi
```

This records a full typescript of every SSH session. Combine with logrotate:

```bash
/etc/logrotate.d/sessions
/var/log/sessions/*.log {
    weekly
    rotate 12
    compress
    delaycompress
    missingok
    notifempty
}
```

fail2ban: Beyond Default SSH Protection

The default fail2ban SSH jail bans after 5 failures. Tighten it:

```ini
/etc/fail2ban/jail.local
[DEFAULT]
bantime  = 3600      # 1 hour (default is 600 seconds)
findtime = 600
maxretry = 3         # reduced from 5

Permanent ban after repeated banning
[recidive]
enabled  = true
filter   = recidive
logpath  = /var/log/fail2ban.log
maxretry = 3
findtime = 86400     # 24 hours
bantime  = 604800    # 7 days

[sshd]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 3600

Block scan tools that send malformed SSH packets
[sshd-ddos]
enabled  = true
port     = ssh
filter   = sshd-ddos
logpath  = /var/log/auth.log
maxretry = 2
bantime  = 7200
```

```bash
sudo systemctl restart fail2ban

Check current bans
sudo fail2ban-client status sshd
sudo fail2ban-client status recidive

Manually unban an IP
sudo fail2ban-client set sshd unbanip 1.2.3.4
```

Port Knocking - Hide SSH from Scanners

Port knocking keeps port 22 closed until a correct sequence of connection attempts unlocks it. Effective against automated scanners.

```bash
Install knockd
sudo apt install knockd

/etc/knockd.conf
[options]
    UseSyslog

[openSSH]
    sequence    = 7000,8000,9000
    seq_timeout = 5
    command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
    tcpflags    = syn

[closeSSH]
    sequence    = 9000,8000,7000
    seq_timeout = 5
    command     = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
    tcpflags    = syn
```

```bash
Default block SSH in iptables
sudo iptables -A INPUT -p tcp --dport 22 -j DROP

sudo systemctl enable --now knockd

To connect from a remote machine:
knock your-server 7000 8000 9000
ssh user@your-server
knock your-server 9000 8000 7000   # close the port again
```

Auditing Current Login Activity

```bash
Who is logged in right now
who
w

Recent successful logins
last -n 20

Recent failed logins
lastb -n 20

All authentication events
sudo journalctl -u ssh --since "24 hours ago" | grep -E "(Accepted|Failed|Invalid)"

IP addresses that attempted login
sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -20
```

Related Articles

- [SSH Server Hardening Config Guide](/ssh-server-hardening-config-guide)
- [How to Harden SSH Server Configuration](/how-to-harden-ssh-server-configuration/)
- [SSH Server Hardening Guide](/ssh-server-hardening-guide/)
- [How to Configure UFW Firewall on Ubuntu](/how-to-configure-ufw-firewall-on-ubuntu/)
- [How to Use YubiKey for SSH Authentication](/articles/how-to-use-yubikey-for-ssh-authentication-guide/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
