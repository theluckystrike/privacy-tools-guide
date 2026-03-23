---
layout: default
title: "How to Use radare2 for Binary Analysis"
description: "Learn radare2 commands for disassembly, function analysis, patching, scripting, and identifying security vulnerabilities in compiled binaries"
date: 2026-03-22
author: theluckystrike
permalink: /radare2-binary-analysis-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# How to Use radare2 for Binary Analysis

radare2 (r2) is a free, open-source reverse engineering framework. It handles disassembly, debugging, binary patching, and scripting across a wide range of architectures and file formats. This guide covers the essential commands for analyzing compiled binaries — finding strings, understanding function logic, identifying vulnerabilities, and extracting useful intelligence.

## Installation

```bash
# Debian/Ubuntu
sudo apt install -y radare2

# macOS
brew install radare2

# Build from source (latest)
git clone https://github.com/radareorg/radare2
cd radare2 && sys/install.sh

# Verify
r2 -v
```

## Opening a Binary

```bash
# Open in read-only mode (safe default)
r2 /path/to/binary

# Open for writing/patching
r2 -w /path/to/binary

# Open and immediately analyze (auto-analysis)
r2 -A /path/to/binary

# Open with debug mode
r2 -d /path/to/binary
```

## Essential Navigation Commands

At the r2 prompt, you issue commands using a consistent syntax: command prefix + subcommand + optional arguments.

```
?           Show help
q           Quit
s <addr>    Seek (move position) to address
s main      Seek to main function
s 0x401000  Seek to specific address
p10         Print 10 bytes at current position
pd 20       Disassemble 20 instructions
pdf         Disassemble current function
```

## Step 1 — Initial Analysis

When you open a binary, run analysis to populate the function database:

```
[0x00000000]> aaa
```

`aaa` runs all analysis passes: function detection, cross-references, strings, symbols. On large binaries it takes a few minutes. For faster partial analysis:

```
aa    # Analyze all (faster)
afl   # List all detected functions
```

View what was found:

```
[0x00000000]> afl
0x00401000  1   6   entry0
0x00401180  3   72  sym.main
0x004011cc  5   104 sym.process_input
...
```

## Step 2 — Analyzing Functions

Seek to main and disassemble it:

```
[0x00000000]> s main
[0x00401180]> pdf
/ (fcn) sym.main 72
|   int main (int argc, char **argv)
|           0x00401180      55             push rbp
|           0x00401181      4889e5         mov rbp, rsp
|           0x00401184      4883ec10       sub rsp, 0x10
|           ...
```

Navigate the function graph visually:

```
[0x00401180]> VV
```

This opens an interactive function graph. Use arrow keys to navigate, `q` to quit back to the command line.

## Step 3 — Strings Analysis

Find embedded strings — useful for spotting hardcoded credentials, API endpoints, and debug messages:

```
[0x00000000]> iz
```

Output:

```
[Strings]
nth paddr      vaddr      len size section type    string
0   0x00402010 0x00402010 24  25   .rodata ascii   "Enter your password: "
1   0x00402029 0x00402029 20  21   .rodata ascii   "Access granted!\n"
2   0x0040203e 0x0040203e 20  21   .rodata ascii   "Access denied!\n"
3   0x00402053 0x00402053 12  13   .rodata ascii   "admin12345\n"
```

The string `admin12345` is a hardcoded credential — a direct security finding from a basic strings scan.

Filter strings by pattern:

```
[0x00000000]> iz~password    # grep-like filter with ~
[0x00000000]> iz~http
[0x00000000]> iz~key
```

## Step 4 — Cross-Reference Analysis

Find where a function is called from (xrefs):

```
[0x00000000]> axt sym.process_input
# Shows all call sites to process_input
```

Find where a string is used:

```
[0x00000000]> /v 0x00402010    # Find references to this address
```

## Step 5 — Finding Common Vulnerabilities

### Unsafe string functions

Search for calls to `strcpy`, `sprintf`, `gets` — classic buffer overflow sources:

```
[0x00000000]> afl~strcpy
[0x00000000]> afl~gets
```

If these functions appear, seek to each call site and examine what precedes the call:

```
[0x00000000]> s sym.imp.strcpy
[0x00401xxx]> axt    # show callers
[0x00000000]> s 0x<caller_address>
[0x00401xxx]> pdf    # disassemble the calling function
```

### Stack canary detection

Check if the binary has security mitigations:

```bash
# From shell (not r2 prompt)
r2 -A /path/to/binary -q -c 'iI' | grep -E "canary|nx|pie|relro"
```

Or in r2:

```
[0x00000000]> iI
```

Look for:
```
canary   true    # Stack canary present
nx       true    # Non-executable stack
pic      true    # Position-independent code
relro    full    # Full RELRO protection
```

A binary with `canary false`, `nx false`, and `pic false` is a classic exploitation target.

## Step 6 — Patching Binaries

Open with write mode to modify bytes directly:

```bash
r2 -w /path/to/binary
```

To patch a conditional jump to always jump (bypass a check):

```
[0x00401200]> pd 5       # View instructions
0x00401200  7405         je 0x00401207    # Jump if equal (zero flag set)

# Patch je to jmp (unconditional)
[0x00401200]> wx eb05    # eb = jmp short, 05 = offset
[0x00401200]> pd 5       # Verify
0x00401200  eb05         jmp 0x00401207
```

## Step 7 — Scripting with r2pipe

Automate analysis with Python via r2pipe:

```bash
pip install r2pipe
```

```python
import r2pipe

r2 = r2pipe.open("/path/to/binary")
r2.cmd("aaa")  # Full analysis

# Get all functions as JSON
functions = r2.cmdj("aflj")
for fn in functions:
    print(f"{fn['name']:40} offset: {hex(fn['offset'])}")

# Find dangerous function calls
dangerous = ["strcpy", "gets", "sprintf", "system"]
for fn in dangerous:
    result = r2.cmd(f"afl~{fn}")
    if result.strip():
        print(f"[!] Found reference to {fn}: {result.strip()}")

r2.quit()
```

```bash
python3 audit_binary.py
```

## Step 8 — Comparing Two Binaries

Useful for patch analysis — finding what changed between firmware versions:

```bash
# Install radiff2 (bundled with radare2)
radiff2 -A /path/to/binary_old /path/to/binary_new

# Compare function-by-function
radiff2 -C /path/to/binary_old /path/to/binary_new
```

## Useful Command Reference

```
Command     Purpose
-------     -------
aaa         Full auto-analysis
afl         List functions
pdf         Disassemble current function
iz          List strings
iI          Binary info (security mitigations)
axt <sym>   Cross-references to symbol
/v <addr>   Find value references
VV          Function graph view
wx <bytes>  Write hex bytes at current position
!r2 -A -q   Run in batch (non-interactive) mode
```

## Related Reading

- [How to Use GDB for Security Debugging](/gdb-security-debugging-guide/)
- [How to Use Ghidra for Reverse Engineering](/ghidra-reverse-engineering-guide/)
- [How to Use binwalk for Firmware Analysis](/binwalk-firmware-analysis-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
