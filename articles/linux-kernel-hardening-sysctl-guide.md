---
layout: default
title: "Linux Kernel Hardening with sysctl"
description: "Harden a Linux server with sysctl settings. Covers network stack hardening, kernel pointer hiding, ASLR, dmesg restriction, and persistence."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: linux-kernel-hardening-sysctl-guide
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

The Linux kernel exposes hundreds of tunable parameters via `/proc/sys/`. Many defaults exist for broad compatibility, not security. Hardening these settings closes off attack paths that local and remote attackers use to escalate privileges, enumerate systems, and exploit the network stack.

This guide covers the most impactful settings with an explanation of what each one does.

## How sysctl Works

Read a current value:

```bash
sysctl kernel.randomize_va_space
# kernel.randomize_va_space = 2
```

Set a value at runtime (does not persist across reboots):

```bash
sudo sysctl -w kernel.randomize_va_space=2
```

Make changes permanent by writing to `/etc/sysctl.d/`:

```bash
sudo nano /etc/sysctl.d/99-hardening.conf
```

Apply all settings from the file:

```bash
sudo sysctl --system
```

## The Complete Hardening File

Create the configuration file:

```bash
sudo nano /etc/sysctl.d/99-hardening.conf
```

Paste the following:

```ini
##############################################################
# KERNEL HARDENING
##############################################################

# Enable ASLR (Address Space Layout Randomization)
# Randomizes memory addresses, making exploitation harder
# 2 = full randomization
kernel.randomize_va_space = 2

# Restrict kernel dmesg access to root
# Hides kernel memory addresses from unprivileged users
kernel.dmesg_restrict = 1

# Restrict access to kernel pointers in /proc
# Hides kernel memory addresses shown in /proc/kallsyms, /proc/modules
kernel.kptr_restrict = 2

# Disable kernel profiling (hides kernel internals from non-root)
kernel.perf_event_paranoid = 3

# Disable kernel magic SysRq key
# Prevents direct kernel reboots/crashes via keyboard
kernel.sysrq = 0

# Restrict use of ptrace to direct parent processes (reduces ptrace abuse)
# 1 = restricted, 2 = admin only, 3 = disabled
kernel.yama.ptrace_scope = 1

# Disable BPF JIT for non-root users (reduces JIT spraying attacks)
kernel.unprivileged_bpf_disabled = 1
net.core.bpf_jit_harden = 2

##############################################################
# NETWORK STACK HARDENING
##############################################################

# Disable IP forwarding (enable ONLY if this is a router)
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# Disable source routing (packets should not dictate their own path)
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# Enable reverse path filtering (prevents IP spoofing)
# Drops packets with source addresses that are unreachable via the incoming interface
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Disable ICMP redirects (attackers use these to redirect traffic)
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Enable TCP SYN cookies (prevents SYN flood DoS attacks)
net.ipv4.tcp_syncookies = 1

# Ignore ICMP broadcast requests (Smurf attack prevention)
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Ignore bogus error messages
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Log martian packets (packets with impossible source addresses)
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Disable secure ICMP redirects (ICMP redirects from gateways)
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0

# Increase TCP connection limits for busy servers
net.ipv4.tcp_max_syn_backlog = 2048

# Reduce TIME_WAIT sockets for faster recovery
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1

# Disable IPv6 if not needed (reduces attack surface)
# Comment out if you use IPv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

# Restrict local port range
net.ipv4.ip_local_port_range = 32768 60999

##############################################################
# FILE SYSTEM HARDENING
##############################################################

# Restrict core dump files (prevents credential leaks from crashed processes)
fs.suid_dumpable = 0

# Protect hardlinks and symlinks from privilege escalation
fs.protected_hardlinks = 1
fs.protected_symlinks = 1

# Restrict /proc/PID/ to process owner
fs.protected_fifos = 2
fs.protected_regular = 2
```

Apply the settings:

```bash
sudo sysctl --system
```

## Verify Key Settings

Check that the most critical settings are applied:

```bash
# ASLR enabled
sysctl kernel.randomize_va_space
# Expected: 2

# Kernel pointers hidden
sysctl kernel.kptr_restrict
# Expected: 2

# dmesg restricted
sysctl kernel.dmesg_restrict
# Expected: 1

# SYN cookies active
sysctl net.ipv4.tcp_syncookies
# Expected: 1

# No ICMP redirects accepted
sysctl net.ipv4.conf.all.accept_redirects
# Expected: 0

# Reverse path filtering
sysctl net.ipv4.conf.all.rp_filter
# Expected: 1
```

## Additional Hardening: Kernel Module Restrictions

Restrict which kernel modules can be loaded. Useful on production servers that shouldn't load new drivers:

```bash
sudo nano /etc/modprobe.d/hardening.conf
```

```
# Disable uncommon network protocols (rarely needed, common attack vector)
install dccp /bin/false
install sctp /bin/false
install rds /bin/false
install tipc /bin/false
install n-hdlc /bin/false
install ax25 /bin/false
install netrom /bin/false
install x25 /bin/false
install rose /bin/false
install decnet /bin/false
install econet /bin/false
install af_802154 /bin/false
install ipx /bin/false
install appletalk /bin/false
install psnap /bin/false
install p8023 /bin/false
install p8022 /bin/false
install can /bin/false
install atm /bin/false

# Disable uncommon filesystems
install cramfs /bin/false
install freevxfs /bin/false
install jffs2 /bin/false
install hfs /bin/false
install hfsplus /bin/false
install squashfs /bin/false
install udf /bin/false

# Disable USB if physical access is not trusted
# install usb-storage /bin/false
```

Update initramfs so changes take effect at boot:

```bash
sudo update-initramfs -u   # Debian/Ubuntu
sudo dracut --force         # Fedora/RHEL
```

## Secure /proc and /sys Mounts

Mount `/proc` with `hidepid=2` so users can only see their own processes:

```bash
sudo nano /etc/fstab
```

Add or modify the proc line:

```
proc  /proc  proc  defaults,hidepid=2,gid=proc  0  0
```

Create a `proc` group for processes that need to see all PIDs (like monitoring tools):

```bash
sudo groupadd -g 1234 proc
sudo usermod -aG proc prometheus   # Add monitoring user to proc group
```

Remount without rebooting:

```bash
sudo mount -o remount,hidepid=2,gid=proc /proc
```

## Audit Current Settings Against Baseline

The `lynis` tool audits Linux security configuration and reports on settings that deviate from hardening benchmarks:

```bash
# Install
sudo apt install lynis   # Debian/Ubuntu
sudo dnf install lynis   # Fedora

# Run audit
sudo lynis audit system

# Look for sysctl suggestions in the report
sudo grep -A2 "sysctl" /var/log/lynis.log | tail -50
```

Lynis outputs a hardening index score and specific recommendations. Run it after applying changes to measure improvement.

## Impact on Performance

Most of these settings have zero measurable performance impact on typical workloads. The exceptions:

- `tcp_syncookies` — adds minor CPU overhead under SYN flood conditions (acceptable tradeoff)
- `log_martians` — generates log entries for spoofed packets (can be noisy on some networks, disable if logs fill up)
- `rp_filter` — can cause issues on multi-homed servers with asymmetric routing (set to `0` per interface if needed)



## Related Articles

- [Arch Linux Hardened Kernel Installation Guide For Advanced P](/privacy-tools-guide/arch-linux-hardened-kernel-installation-guide-for-advanced-p/)
- [Linux Desktop Privacy Hardening Guide](/privacy-tools-guide/linux-desktop-privacy-hardening-guide/)
- [Complete Guide To Operating System Hardening For Extreme Pri](/privacy-tools-guide/complete-guide-to-operating-system-hardening-for-extreme-pri/)
- [SSH Server Hardening Config Guide](/privacy-tools-guide/ssh-server-hardening-config-guide)
- [Best Password Manager for Linux in 2026: A Developer's Guide](/privacy-tools-guide/best-password-manager-for-linux/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
