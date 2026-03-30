---
layout: default
title: "How to Use Volatility for Memory Forensics"
description: "Analyze RAM dumps with Volatility 3 to recover running processes, network connections, injected code, and malware artifacts from Windows and Linux memor..."
date: 2026-03-22
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /volatility-memory-forensics-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
How to Use Volatility for Memory Forensics

Volatility is the standard framework for analyzing memory dumps. While disk forensics tells you what was stored, memory forensics tells you what was actively running. including fileless malware, injected shellcode, and processes that have already deleted themselves from disk. This guide covers Volatility 3 on Linux, analyzing Windows and Linux memory images.

What You Can Extract from RAM

- Running processes (including hidden/injected ones)
- DLLs and modules loaded by each process
- Open network connections with remote IPs and ports
- Registry hives loaded in memory
- Command-line arguments of processes
- Encryption keys (BitLocker, TrueCrypt) if they were active
- Browser credentials cached in memory
- Injected shellcode via VAD (Virtual Address Descriptor) analysis

---

1. Install Volatility 3

```bash
Ubuntu / Debian
sudo apt install -y python3 python3-pip python3-venv git

python3 -m venv ~/vol3env
source ~/vol3env/bin/activate

pip install volatility3

Or from source for latest plugins
git clone https://github.com/volatilityfoundation/volatility3.git ~/volatility3
cd ~/volatility3
pip install -r requirements.txt
python3 vol.py --version
```

---

2. Acquire a Memory Dump

From a running Linux system (for your own forensics lab):

```bash
Using LiME (Linux Memory Extractor)
sudo apt install -y build-essential linux-headers-$(uname -r)

git clone https://github.com/504ensicsLabs/LiME.git /tmp/lime
cd /tmp/lime/src && make

Load module and dump to file
sudo insmod lime-$(uname -r).ko "path=/tmp/memory.lime format=lime"
ls -lh /tmp/memory.lime
```

From a running Windows system (authorized investigation only):

Use WinPmem - `winpmem_mini_x64.exe memory.raw`

From a VM snapshot:

VirtualBox: `VBoxManage debugvm <vmname> dumpvmcore --filename memory.elf`
VMware - The `.vmem` file in the VM directory is the raw memory dump.

---

3. Identify the Profile / Symbol Table

Volatility 3 uses symbol tables instead of profiles. For Windows, it auto-detects the OS version. For Linux, you need a symbol table matching the exact kernel version.

```bash
vol.py -f memory.lime windows.info
OR
vol.py -f memory.raw imageinfo   # older compat command

For Linux images, generate or download a symbol table:
dwarf2json (converts kernel dwarf info to Volatility symbol table)
git clone https://github.com/volatilityfoundation/dwarf2json.git
cd dwarf2json && go build

./dwarf2json linux --elf /boot/vmlinux-$(uname -r) > \
  ~/volatility3/volatility3/symbols/linux/$(uname -r).json.xz
```

---

4. List Running Processes

```bash
Windows memory
vol.py -f memory.raw windows.pslist

Linux memory
vol.py -f memory.lime linux.pslist

pstree. shows parent-child relationships (good for spotting unusual spawns)
vol.py -f memory.raw windows.pstree

psscan. finds processes by scanning memory, catches hidden processes
that are unlinked from the process list
vol.py -f memory.raw windows.psscan
```

Compare `pslist` (walks the process list) with `psscan` (scans all memory). Processes in `psscan` but not in `pslist` are hidden. a strong malware indicator.

---

5. Inspect Network Connections

```bash
Windows. active and recently closed connections
vol.py -f memory.raw windows.netscan

Linux
vol.py -f memory.lime linux.netstat

Windows XP (older format)
vol.py -f memory.raw windows.connections
```

Example output:

```
Offset    Proto  LocalAddr          ForeignAddr        State    PID  Owner
0xfa8...  TCPv4  192.168.1.10:443   203.0.113.5:54321  CLOSE_W  1234 chrome.exe
0xfa9...  TCPv4  0.0.0.0:4444       0.0.0.0:0          LISTEN   5678 svchost.exe
```

Port 4444 listening from `svchost.exe` is a Metasploit default. investigate further.

---

6. Detect Code Injection (malfind)

`malfind` scans VAD entries for memory regions that are executable, writeable, and not backed by a file on disk. the classic signature of injected shellcode:

```bash
vol.py -f memory.raw windows.malfind

Dump suspicious regions to files for further analysis
vol.py -f memory.raw windows.malfind --dump --dump-dir /tmp/injected/

ls /tmp/injected/
pid.1234.vad.0x400000.dmp
pid.1234.vad.0x500000.dmp
```

Analyze dumped regions with YARA or upload to VirusTotal:

```bash
Scan with YARA rules
yara /path/to/rules.yar /tmp/injected/ -r

Get hash for VirusTotal lookup
sha256sum /tmp/injected/pid.1234.vad.0x400000.dmp
```

---

7. List DLLs for a Specific Process

```bash
List all DLLs loaded by process ID 1234
vol.py -f memory.raw windows.dlllist --pid 1234

Spot DLLs loaded from temp directories or unusual paths
vol.py -f memory.raw windows.dlllist | grep -i "\\\\temp\\\\"
vol.py -f memory.raw windows.dlllist | grep -i "\\\\appdata\\\\"
```

---

8. Dump a Specific Process

```bash
Dump entire process memory space
vol.py -f memory.raw windows.memmap --pid 1234 --dump --dump-dir /tmp/proc_1234/

Dump just the executable
vol.py -f memory.raw windows.dumpfiles --pid 1234 --dump-dir /tmp/proc_1234/

Strings analysis of the process dump
strings /tmp/proc_1234/*.dmp | grep -iE "http|ftp|cmd|powershell|base64"
```

---

9. Windows Registry Hives in Memory

```bash
List registry hives loaded in memory
vol.py -f memory.raw windows.registry.hivelist

Print key/value from a specific hive
vol.py -f memory.raw windows.registry.printkey \
  --key "Software\\Microsoft\\Windows\\CurrentVersion\\Run"

Common persistence locations to check
vol.py -f memory.raw windows.registry.printkey \
  --key "SYSTEM\\CurrentControlSet\\Services"
```

---

10. Detect Kernel Rootkits (SSDT Hooks)

Rootkits modify the System Service Descriptor Table (SSDT) to intercept system calls:

```bash
Check for SSDT hooks (Windows)
vol.py -f memory.raw windows.ssdt

Any entry not pointing into ntoskrnl.exe or win32k.sys is suspicious
vol.py -f memory.raw windows.ssdt | grep -v "ntoskrnl\|win32k"

Check for hidden kernel modules
vol.py -f memory.raw windows.modules   # official list
vol.py -f memory.raw windows.modscan   # full scan
Entries in modscan but not modules = hidden driver
```

---

11. Extract Password Hashes

```bash
Windows SAM hashes (if you have SYSTEM + SAM hive offsets)
vol.py -f memory.raw windows.hashdump

LSASS process contains NTLM hashes in memory
vol.py -f memory.raw windows.lsadump

Extract cached credentials
vol.py -f memory.raw windows.cachedump
```

---

Automated Triage Script

```python
#!/usr/bin/env python3
"""
Quick triage - run key Volatility plugins and save output.
Usage - python3 triage.py memory.raw /tmp/triage_output/
"""
import subprocess, sys, os

DUMP    = sys.argv[1]
OUTDIR  = sys.argv[2]
os.makedirs(OUTDIR, exist_ok=True)

PLUGINS = [
    "windows.pslist", "windows.psscan", "windows.pstree",
    "windows.netscan", "windows.malfind",
    "windows.dlllist", "windows.registry.hivelist",
    "windows.modules", "windows.modscan",
]

for plugin in PLUGINS:
    outfile = os.path.join(OUTDIR, plugin.replace(".", "_") + ".txt")
    cmd = ["vol.py", "-f", DUMP, plugin]
    print(f"[*] Running {plugin}...")
    with open(outfile, "w") as f:
        subprocess.run(cmd, stdout=f, stderr=subprocess.DEVNULL)
    print(f"    Saved → {outfile}")

print("\n[+] Triage complete.")
```

---

Related Reading

- [How to Use Cuckoo Sandbox for Malware Analysis](/cuckoo-sandbox-malware-analysis-guide/)
- [How to Set Up Snort IDS on Linux](/snort-ids-linux-setup-guide/)
- [How to Use Zeek for Network Monitoring](/zeek-network-monitoring-guide/)
- [1Password Families Plan Review 2026: Is It Worth It](/1password-families-plan-review-2026/)
- [AI Coding Assistant Session Data Lifecycle](https://bestremotetools.com/ai-coding-assistant-session-data-lifecycle-from-request-to-deletion-explained-2026/)

---

Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
