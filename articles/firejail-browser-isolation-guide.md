---
layout: default
title: "Browser Isolation with Firejail Guide"
description: "Sandbox Firefox and Chromium with Firejail on Linux. Custom profiles, private network namespaces, and seccomp filters to contain breaches."
date: 2026-03-22
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /firejail-browser-isolation-guide/
categories: [guides, security]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Browser Isolation with Firejail Guide

Firejail is a SUID sandbox that restricts what processes can access using Linux namespaces, seccomp-bpf, and capabilities. When your browser is sandboxed with Firejail, a successful exploit. remote code execution via a malicious PDF or JavaScript vulnerability. is contained: the attacker's code can't read your SSH keys, home directory files, or make arbitrary system calls.
---

What Firejail Does

Firejail creates restricted containers using:

- Linux namespaces: Separate filesystem view, process table, network stack, and user namespace
- seccomp-bpf: Whitelist of allowed system calls. unknown/dangerous calls are blocked
- Capabilities - Drop Linux capabilities the application doesn't need (e.g., no raw socket access)
- AppArmor (optional): Additional MAC policy layer

The result - a browser running under Firejail sees a restricted subset of your filesystem and can only make system calls that are in the allowed list.

---

Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


Step 1 - Install Firejail

```bash
Debian/Ubuntu
sudo apt install firejail firejail-profiles

Fedora
sudo dnf install firejail

Arch Linux
sudo pacman -S firejail

Check version
firejail --version

List available profiles
ls /etc/firejail/*.profile | head -20
```

---

Step 2 - Basic Usage

```bash
Run Firefox in a Firejail sandbox
firejail firefox

Run Chromium sandboxed
firejail chromium

Run with default profile for the application
firejail --profile=/etc/firejail/firefox.profile firefox

Inspect what's running inside Firejail
firejail --list

See the security details of a running sandboxed process
firejail --tree

Get detailed security status of a specific PID
firejail --status
```

---

Step 3 - Desktop Integration (Launch From App Menu)

To always launch Firefox sandboxed, create a desktop entry override:

```bash
Copy system desktop entry to user override directory
cp /usr/share/applications/firefox.desktop ~/.local/share/applications/

Edit the Exec line
nano ~/.local/share/applications/firefox.desktop
```

Change `Exec=firefox %u` to `Exec=firejail firefox %u`:

```ini
[Desktop Entry]
Name=Firefox
Exec=firejail firefox %u
Icon=firefox
Type=Application
Categories=Network;WebBrowser;
MimeType=text/html;text/xml;...
```

```bash
Refresh desktop database
update-desktop-database ~/.local/share/applications/

Or use firecfg to automate this for all supported apps
sudo firecfg
Creates symlinks in /usr/local/bin/ that wrap apps with firejail
```

---

Step 4 - Understand Default Firefox Profile

Examine the default Firejail Firefox profile:

```bash
cat /etc/firejail/firefox.profile
```

Key restrictions applied:

```ini
From firefox.profile:
include globals.local

Private home directory (creates a temporary home)
private-home firefox-home

Whitelist specific directories
whitelist ${HOME}/Downloads
whitelist ${HOME}/.mozilla

Block sensitive paths
blacklist ${HOME}/.ssh
blacklist ${HOME}/.gnupg
blacklist ${HOME}/.config/keepassxc

Network restrictions
(by default, network access is unrestricted)

Disable D-Bus (prevents IPC with system)
nodbus

Drop dangerous capabilities
caps.drop all
caps.keep net_bind_service

Seccomp filter (deny dangerous syscalls)
seccomp
```

---

Step 5 - Custom Profile: Hardened Firefox

Create a stricter profile for maximum isolation:

```bash
Create custom profile
nano ~/.config/firejail/firefox.local
```

```ini
~/.config/firejail/firefox.local
This file is included by /etc/firejail/firefox.profile

Block clipboard access to/from system (prevents clipboard snooping)
Remove if you need copy-paste between sandboxed and non-sandboxed apps
x11

Restrict file access to only Downloads
private-home firefox-sandbox
whitelist ${HOME}/Downloads
whitelist ${HOME}/.mozilla

Block access to other user data
blacklist ${HOME}/Documents
blacklist ${HOME}/Pictures
blacklist ${HOME}/.ssh
blacklist ${HOME}/.aws
blacklist ${HOME}/.config/keepassxc
blacklist ${HOME}/.local/share/keyrings

Disable sound if not needed (reduces attack surface)
nosound

Use a private /tmp (prevents /tmp sniffing)
private-tmp

Private network namespace - only allow specific interfaces
net eth0   # Uncomment to restrict to specific interface
```

---

Step 6 - Network Isolation for Sandboxed Apps

Firejail can create a completely separate network namespace:

```bash
Run browser with isolated network (requires network bridge setup)
firejail --net=eth0 firefox

Run with no network access at all
firejail --net=none firefox   # Useful for testing or offline use

Run on a VPN interface only (forces all traffic through VPN)
firejail --net=wg0 firefox
If VPN drops, browser loses network entirely (natural kill switch)

Create a network sandbox that can only reach Tor
firejail --net=lo --netfilter firefox
Then use tor's SOCKS proxy at 127.0.0.1:9050
```

---

Step 7 - Sandbox Testing: Verify What's Blocked

```bash
Run a test to see what Firejail blocks
firejail --debug bash

Inside the sandbox, try to access restricted paths:
cat ~/.ssh/id_rsa        # Should fail: No such file or directory
ls ~/Documents           # Should fail if blacklisted

Check what filesystem the sandbox sees
mount | grep tmpfs       # Shows private mounts created by Firejail

Check system call filtering
strace -e trace=all ls 2>&1 | head -20
Certain syscalls will show EPERM if blocked by seccomp
```

---

Step 8 - Firejail for Other Applications

```bash
Sandbox a PDF reader (high-value target for malicious PDFs)
firejail evince malicious-document.pdf
firejail okular research-paper.pdf

Sandbox media players
firejail vlc downloaded-video.mp4

Sandbox email client
firejail thunderbird

Sandbox an unknown binary (testing)
firejail --net=none --private ./unknown-program
No network + private home = maximum isolation for unknown executables
```

---

Step 9 - Audit Firejail's Effectiveness

```bash
Check what seccomp filters are applied
firejail --debug --seccomp firefox 2>&1 | grep "seccomp"

See the full security report for a running sandboxed process
firejail --list
Get the PID, then:
firejail --status [PID]

Try to escape from inside a sandbox (penetration testing)
Inside sandboxed bash:
ls /proc/1/root     # Should be restricted
cat /etc/shadow     # Should fail
cat ~/.ssh/id_rsa   # Should fail if blacklisted
```

---

Combining Firejail with AppArmor

For double-layered isolation, combine Firejail (process-level) with AppArmor (MAC policy):

```bash
Check AppArmor status
sudo aa-status | grep -i firefox

Firejail's AppArmor profile is auto-applied if AppArmor is active
firejail --apparmor firefox

Create a tight AppArmor profile for the sandboxed browser
sudo aa-genprof firejail
Follow prompts to define allowed behavior
sudo systemctl reload apparmor
```

---

Performance Impact

Firejail adds minimal CPU overhead (< 1% for most apps). The main cost is startup time:

```bash
Compare startup times
time firejail firefox --headless --window-size 1 1 &
time firefox --headless --window-size 1 1 &
Difference is typically 0.1-0.5 seconds for namespace setup
```

For GPU-accelerated applications (browsers, video players), Firejail requires DRI device access:

```bash
Allow DRI (GPU) access in profile:
In your local profile file:
whitelist /dev/dri
noblacklist /dev/dri
```

---

Troubleshooting

Configuration changes not taking effect

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

Permission denied errors

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

Connection or network-related failures

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


Related Articles

- [Tor Browser Isolation Container Setup Guide](/tor-browser-isolation-container-setup-guide/)
- [Browser First-Party Isolation: What It Does and How It Works](/browser-first-party-isolation-what-it-does/)
- [Tor Browser Cookies Tracking Prevention Guide](/tor-browser-cookies-tracking-prevention-guide/)
- [Browser Storage Isolation Explained](/browser-storage-isolation-explained-privacy/)
- [Tor Browser Android Setup Guide](/tor-browser-android-setup-guide-orbot/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)
Built by theluckystrike. More at [zovo.one](https://zovo.one)

Frequently Asked Questions

How long does it take to complete this setup?

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

What are the most common mistakes to avoid?

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

Do I need prior experience to follow this guide?

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

Can I adapt this for a different tech stack?

Yes, the underlying concepts transfer to other stacks, though the specific implementation details will differ. Look for equivalent libraries and patterns in your target stack. The architecture and workflow design remain similar even when the syntax changes.

Where can I get help if I run into issues?

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.
{% endraw %}
