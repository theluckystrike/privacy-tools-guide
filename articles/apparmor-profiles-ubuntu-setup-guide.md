---
layout: default
title: "How to Set Up AppArmor Profiles on Ubuntu"
description: "Create, test, and enforce AppArmor profiles to confine applications on Ubuntu, including custom profiles for web servers, databases, and containers"
date: 2026-03-22
author: theluckystrike
permalink: /apparmor-profiles-ubuntu-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

How to Set Up AppArmor Profiles on Ubuntu

AppArmor restricts what a process can do by defining explicit rules about which files it can read, write, or execute, which network operations it can perform, and which capabilities it can use. When a program is compromised, AppArmor limits what the attacker can reach. Ubuntu ships with AppArmor enabled and includes profiles for common daemons. this guide covers writing custom profiles and enforcing them.

Quick Setup Steps

1. Check AppArmor status: `sudo aa-status` to see loaded profiles and their modes
2. Install utilities: `sudo apt install apparmor-utils apparmor-profiles apparmor-profiles-extra`
3. Generate a profile skeleton: `sudo aa-genprof /path/to/application`
4. Run the application in another terminal while aa-genprof monitors its behavior
5. Review and approve access rules as aa-genprof presents them interactively
6. Test in complain mode: `sudo aa-complain /etc/apparmor.d/your.profile` to log violations without blocking
7. Check logs for denials: `sudo dmesg | grep apparmor` or `journalctl -k | grep apparmor`
8. Enforce the profile: `sudo aa-enforce /etc/apparmor.d/your.profile`
9. Reload after edits: `sudo apparmor_parser -r /etc/apparmor.d/your.profile`


Understanding AppArmor Modes

AppArmor runs profiles in two modes:
- enforce. violations are blocked and logged
- complain. violations are only logged (learning mode. use this to build a profile)

```bash
Check AppArmor status
sudo aa-status

Typical output:
34 profiles are loaded.
32 profiles are in enforce mode.
2 profiles are in complain mode.
4 processes have profiles defined.
```

Tools You Need

```bash
sudo apt install apparmor-utils apparmor-profiles apparmor-profiles-extra
```

Key commands:
- `aa-genprof`. interactive profile generator
- `aa-logprof`. update a profile from audit logs
- `aa-enforce`. put a profile into enforce mode
- `aa-complain`. put a profile into complain mode
- `aa-disable`. disable a profile
- `aa-status`. show all profiles and their mode

Step 1 - Generate a Profile with aa-genprof

`aa-genprof` runs your application, watches what it does, and builds an initial profile.

```bash
create a profile for a custom Python script
sudo aa-genprof /usr/local/bin/myapp.py
```

`aa-genprof` will prompt you to run the application in another terminal:

```bash
Terminal 2 - run the application through its normal operations
python3 /usr/local/bin/myapp.py --config /etc/myapp/config.yaml
```

Back in terminal 1, press `S` to scan the log, then review each access and allow or deny it. At the end, press `F` to finish and save.

Step 2 - Write a Profile Manually

Understanding profile syntax lets you write precise rules.

```
/etc/apparmor.d/usr.local.bin.myapp
AppArmor profile for myapp

#include <tunables/global>

/usr/local/bin/myapp.py {
  #include <abstractions/base>
  #include <abstractions/python>

  # Binary itself (read and execute)
  /usr/local/bin/myapp.py r,
  /usr/bin/python3* rix,

  # Config file (read only)
  /etc/myapp/config.yaml r,

  # Data directory (read and write)
  /var/lib/myapp/ r,
  /var/lib/myapp/ rw,

  # Log file (append only)
  /var/log/myapp.log a,

  # PID file
  /run/myapp.pid rw,

  # Deny access to sensitive paths explicitly
  deny /etc/shadow r,
  deny /etc/passwd r,
  deny /root/ rw,
  deny /home/ rw,

  # Network: allow outbound TCP on port 443 only
  network inet stream,
  network inet6 stream,

  # Capabilities needed (list only what's required)
  # capability net_bind_service,   # only if binding to port < 1024
}
```

Permission flags:
- `r`. read
- `w`. write
- `x`. execute
- `a`. append
- `l`. link
- `rix`. read, inherit, execute (child inherits parent profile)
- `Px`. execute with a specific profile

Step 3 - Profile for nginx

```
/etc/apparmor.d/usr.sbin.nginx
#include <tunables/global>

/usr/sbin/nginx {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  # nginx binary
  /usr/sbin/nginx mr,

  # Configuration
  /etc/nginx/ r,
  /etc/nginx/ r,

  # Certificates
  /etc/ssl/certs/ r,
  /etc/ssl/certs/ r,
  /etc/letsencrypt/live/ r,
  /etc/letsencrypt/live/ r,
  /etc/letsencrypt/archive/ r,
  /etc/letsencrypt/archive/ r,

  # Web root
  /var/www/ r,
  /var/www/ r,

  # Logs
  /var/log/nginx/ r,
  /var/log/nginx/ rw,

  # Runtime
  /run/nginx.pid rw,
  /tmp/nginx-* rw,

  # Network
  network inet stream,
  network inet6 stream,
  network unix stream,

  # Capabilities
  capability net_bind_service,
  capability setuid,
  capability setgid,
  capability dac_override,

  # Deny access to anything else
  deny /etc/shadow r,
  deny /proc/*/mem r,
  deny @{HOME}/ rw,
}
```

Step 4 - Load and Enforce a Profile

```bash
Parse and load a profile
sudo apparmor_parser -r /etc/apparmor.d/usr.local.bin.myapp

Start in complain mode first
sudo aa-complain /usr/local/bin/myapp.py

Run the application and check logs
sudo journalctl -f -k | grep apparmor &
/usr/local/bin/myapp.py --test-all-features

Review what was denied
sudo cat /var/log/syslog | grep DENIED | tail -30

Update profile from logs
sudo aa-logprof

Once happy, enforce
sudo aa-enforce /usr/local/bin/myapp.py

Reload
sudo apparmor_parser -r /etc/apparmor.d/usr.local.bin.myapp
sudo systemctl restart myapp
```

Step 5 - Debugging Denials

```bash
Watch for AppArmor denials in real time
sudo journalctl -f | grep 'apparmor="DENIED"'

Typical denial log entry:
kernel: audit: type=1400 audit(1711040000.123:456): apparmor="DENIED"
operation="open" profile="/usr/local/bin/myapp.py"
name="/etc/hosts" pid=12345 comm="python3"
requested_mask="r" denied_mask="r" fsuid=1000 ouid=0

Parse AppArmor audit logs with aa-logprof
sudo aa-logprof

It reads /var/log/syslog or /var/log/audit/audit.log
and presents each denial for you to allow/deny
```

When you see a denial that should be allowed, add the rule to the profile:

```bash
sudo nano /etc/apparmor.d/usr.local.bin.myapp
Add the missing rule, e.g. - /etc/hosts r,
sudo apparmor_parser -r /etc/apparmor.d/usr.local.bin.myapp
```

Step 6 - AppArmor and Docker

Docker containers can have AppArmor profiles applied:

```bash
Run a container with a custom AppArmor profile
docker run --security-opt apparmor=docker-nginx-profile nginx

List active container profiles
docker inspect my-container | jq '.[0].HostConfig.SecurityOpt'
```

Docker includes a default AppArmor profile (`docker-default`) that is applied automatically. You can inspect it:

```bash
cat /etc/apparmor.d/docker-default
```

To customize it for a specific service, copy and modify the default profile, then reference it by name.

Auditing Profiles with aa-status

```bash
Full status
sudo aa-status

Just the enforced profiles
sudo aa-status | grep "enforce mode" -A 50 | grep "   /"

Check which processes are confined
sudo aa-status --json | jq '.processes | to_entries[] | {profile: .key, pids: .value}'
```

Related Articles

- [Create Separate Browser Profiles For Each Online Identity](/how-to-create-separate-browser-profiles-for-each-online-identity-compartmentalization/)
- [Android Work Profile for Isolating Apps That Require](/android-work-profile-for-isolating-apps-that-require-invasiv/)
- [How To Verify Dating Profile Authenticity Without Revealing](/how-to-verify-dating-profile-authenticity-without-revealing-/)
- [How To Set Up Mobile Device Management Profile For Personal](/how-to-set-up-mobile-device-management-profile-for-personal-/)
- [How To Prevent Expartner From Creating Fake Dating Profiles](/how-to-prevent-expartner-from-creating-fake-dating-profiles-/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
