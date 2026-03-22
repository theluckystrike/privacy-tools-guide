---
layout: default
title: "How to Configure SELinux Policies Step by Step"
description: "Practical guide to writing, auditing, and enforcing custom SELinux policies on RHEL and Ubuntu using audit2allow, semanage, and policy modules"
date: 2026-03-22
author: theluckystrike
permalink: /selinux-policy-configuration-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
# How to Configure SELinux Policies Step by Step

SELinux (Security-Enhanced Linux) enforces mandatory access control at the kernel level. Even if an attacker gains code execution as root, SELinux can prevent them from reading sensitive files or spawning network connections outside a tightly defined policy. This guide covers real-world policy management on RHEL 9 / Fedora 40 and Ubuntu 24.04 with the `selinux-utils` stack.

## Understanding SELinux Modes

```bash
# Check current mode
getenforce           # Enforcing | Permissive | Disabled

# Check per-file mode and policy type
sestatus

# Switch modes without rebooting
sudo setenforce 1    # Enforcing
sudo setenforce 0    # Permissive (logs but doesn't block)
```

**Enforcing** — denials are enforced. **Permissive** — denials are logged only; use this while building policy. **Disabled** requires a reboot and relabeling.

Permanent mode is set in `/etc/selinux/config`:

```bash
SELINUX=enforcing
SELINUXTYPE=targeted   # targeted (most systems) or mls (high-security)
```

---

## Install Tools

```bash
# RHEL / Fedora
sudo dnf install -y policycoreutils policycoreutils-python-utils \
  setools-console setroubleshoot-server audit

# Ubuntu / Debian
sudo apt install -y selinux-basics selinux-policy-default \
  selinux-utils policycoreutils auditd
sudo selinux-activate   # enables SELinux; requires reboot
```

---

## Understanding Contexts

Every file, process, and network port has a label: `user:role:type:level`.

```bash
# View file context
ls -Z /etc/passwd
# system_u:object_r:passwd_file_t:s0

# View process context
ps auxZ | grep nginx
# system_u:system_r:httpd_t:s0

# View port contexts
sudo semanage port -l | grep http
# http_port_t   tcp   80, 443, 8008, 8080, 8443
```

The `type` field (e.g., `httpd_t`, `sshd_t`) is the core enforcement unit. A process running with type `httpd_t` can only access types that policy explicitly allows.

---

## Step 1: Identify Denials

When SELinux blocks an action, it writes to the audit log:

```bash
sudo ausearch -m AVC -ts recent          # last 10 minutes
sudo ausearch -m AVC -ts today           # today's denials
sudo ausearch -m AVC -c nginx            # denials for nginx only

# Human-readable explanation
sudo sealert -a /var/log/audit/audit.log | less
```

A typical AVC denial:

```
type=AVC msg=audit(1711100400.001:1234): avc: denied { read } for
  pid=12345 comm="nginx" name="app.sock"
  scontext=system_u:system_r:httpd_t:s0
  tcontext=system_u:object_r:user_tmp_t:s0 tclass=sock_file permissive=0
```

This tells you: nginx (httpd_t) was denied **read** access to a socket labeled **user_tmp_t**.

---

## Step 2: Fix with Boolean or Context Change

Many common scenarios are already covered by **booleans** — toggle switches for pre-written policy branches:

```bash
# List all booleans
getsebool -a

# Allow nginx to connect to any port (reverse proxy use case)
sudo setsebool -P httpd_can_network_connect on

# Allow nginx to serve user home directories
sudo setsebool -P httpd_enable_homedirs on

# Allow HTTPD scripts to connect to databases
sudo setsebool -P httpd_can_network_connect_db on
```

The `-P` flag makes the change persistent across reboots.

For a file context mismatch, relabel the file rather than writing a new policy:

```bash
# Check what context a path should have
matchpathcon /var/www/myapp/data

# Restore default context for a path
sudo restorecon -Rv /var/www/myapp/

# Permanently add a custom context rule
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/myapp/data(/.*)?"
sudo restorecon -Rv /var/www/myapp/data
```

---

## Step 3: Allow a Non-Standard Port

If your application listens on a non-standard port (e.g., Nginx on 8888):

```bash
# Check if port is already labeled
sudo semanage port -l | grep 8888

# Add the label
sudo semanage port -a -t http_port_t -p tcp 8888

# Verify
sudo semanage port -l | grep 8888
# http_port_t   tcp   80, 443, 8008, 8080, 8443, 8888
```

---

## Step 4: Write a Custom Policy Module with audit2allow

When booleans and context changes don't cover your case, generate a custom policy module from audit logs:

```bash
# Run your application in permissive mode and exercise all code paths
# Then extract relevant denials
sudo ausearch -m AVC -ts today -c myapp > /tmp/myapp_denials.log

# Convert to a policy module
sudo audit2allow -M myapp_policy < /tmp/myapp_denials.log

# This generates:
#   myapp_policy.te   (human-readable type enforcement source)
#   myapp_policy.pp   (compiled binary policy module)

# Review the .te file before loading
cat myapp_policy.te
```

A generated `.te` file looks like:

```
module myapp_policy 1.0;

require {
    type httpd_t;
    type user_tmp_t;
    class sock_file read;
    class unix_stream_socket connectto;
}

#============= httpd_t ==============
allow httpd_t user_tmp_t:sock_file read;
allow httpd_t user_tmp_t:unix_stream_socket connectto;
```

If this looks correct, load it:

```bash
sudo semodule -i myapp_policy.pp

# Verify it's loaded
sudo semodule -l | grep myapp
```

---

## Step 5: Write Policy from Scratch (Advanced)

For applications that need a fully isolated domain, write a `.te` file manually:

```
# /etc/selinux/targeted/src/myservice.te
policy_module(myservice, 1.0)

type myservice_t;
type myservice_exec_t;

init_daemon_domain(myservice_t, myservice_exec_t)

# Allow reading its own config
allow myservice_t myservice_t:file { read open getattr };

# Allow binding to a specific port (must be labeled first with semanage)
allow myservice_t myservice_port_t:tcp_socket name_bind;

# Allow logging to /var/log
logging_write_generic_logs(myservice_t)
```

```bash
# Compile and install
make -f /usr/share/selinux/devel/Makefile myservice.pp
sudo semodule -i myservice.pp

# Label the binary
sudo semanage fcontext -a -t myservice_exec_t /usr/local/bin/myservice
sudo restorecon /usr/local/bin/myservice
```

---

## Step 6: Audit and Harden Existing Policy

```bash
# Show all rules for a type
sesearch --allow -s httpd_t | grep write

# Find all types a domain can access
sesearch --allow -s httpd_t -c file

# List all domains that can write to a sensitive type
sesearch --allow -t shadow_t -p write

# Detect policy violations via setroubleshoot
sudo journalctl -u setroubleshootd -f
```

---

## Step 7: Apply Labels in Dockerfile / Ansible

For containerized workloads or automated provisioning:

```yaml
# Ansible task to set file context
- name: Set SELinux context for app data
  community.general.sefcontext:
    target: '/opt/myapp/data(/.*)?'
    setype: httpd_sys_rw_content_t
    state: present

- name: Apply context
  ansible.builtin.command: restorecon -Rv /opt/myapp/data
```

In Podman/Docker, pass the `:Z` or `:z` bind-mount flag to auto-label volumes:

```bash
podman run -v /data/app:/app:Z myimage   # :Z = private label for this container
podman run -v /data/shared:/app:z myimage # :z = shared label (all containers)
```

---

## Debugging Checklist

| Symptom | Action |
|---|---|
| Application fails silently | Check `ausearch -m AVC -ts recent` |
| Denial log is empty | Confirm audit daemon is running: `systemctl status auditd` |
| restorecon has no effect | Check if parent dir has wrong context first |
| Module loads but denials continue | Ensure no `dontaudit` rule is hiding them: `semodule -DB` then retry |
| Entire system blocks boot | Add `enforcing=0` to kernel cmdline to boot permissive |

---

## Related Reading

- [How to Set Up Snort IDS on Linux](/privacy-tools-guide/snort-ids-linux-setup-guide/)
- [AppArmor Profiles Ubuntu Setup Guide](/privacy-tools-guide/apparmor-profiles-ubuntu-setup-guide/)
- [How to Set Up OpenSCAP for Compliance Scanning](/privacy-tools-guide/openscap-compliance-scanning-setup-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
