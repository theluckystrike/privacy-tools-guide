---
layout: default
title: "How to Use GDB for Security Debugging"
description: "Use GDB to analyze program memory, find buffer overflows, inspect function calls, and understand exploitable conditions in compiled Linux binaries"
date: 2026-03-22
author: theluckystrike
permalink: /gdb-security-debugging-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, troubleshooting, security]
---

{% raw %}

# How to Use GDB for Security Debugging

GDB (GNU Debugger) is the standard debugger for Linux binaries. Security researchers use it to step through execution, inspect memory, find overflow conditions, and understand what a suspicious binary actually does at runtime. This guide focuses on security-relevant techniques: stack analysis, vulnerability identification, and controlled execution.

## Installation

```bash
# Debian/Ubuntu
sudo apt install -y gdb gdb-multiarch

# Add GDB extensions for better security research experience
# PEDA (Python Exploit Development Assistance)
git clone https://github.com/longld/peda.git ~/peda
echo "source ~/peda/peda.py" >> ~/.gdbinit

# Or pwndbg (more active development)
git clone https://github.com/pwndbg/pwndbg
cd pwndbg && ./setup.sh
```

Verify:

```bash
gdb --version
```

## Starting GDB

```bash
# Debug a binary
gdb ./target_binary

# Debug with arguments
gdb --args ./target_binary arg1 arg2

# Attach to running process
gdb -p PID

# Remote debugging (target running GDB server)
gdb target_binary
(gdb) target remote 192.168.1.50:1234
```

## Essential GDB Commands

```
Command             Action
-------             ------
r / run             Start execution
r arg1 arg2         Start with arguments
c / continue        Continue execution after breakpoint
s / step            Step into next instruction (source level)
si                  Step into next instruction (assembly level)
n / next            Step over next instruction (source level)
ni                  Step over next instruction (assembly level)
q / quit            Exit GDB
```

## Setting Breakpoints

```bash
# Break at function name
(gdb) break main
(gdb) break sym.process_input

# Break at address
(gdb) break *0x00401180

# Break at line number (if debug symbols available)
(gdb) break main.c:42

# List all breakpoints
(gdb) info breakpoints

# Delete breakpoint
(gdb) delete 1
```

## Inspecting Memory and Registers

```bash
# Show all register values
(gdb) info registers

# Show specific register
(gdb) print $rsp
(gdb) print $rbp
(gdb) print $rip

# Examine memory at address
# Format: x/[count][format][size] address
# Formats: x=hex, d=decimal, s=string, i=instructions
# Sizes: b=byte, h=halfword, w=word, g=giant(8bytes)

# 16 hex bytes at stack pointer
(gdb) x/16xb $rsp

# 20 instructions at instruction pointer
(gdb) x/20i $rip

# String at address
(gdb) x/s 0x402010

# Stack inspection — 20 words from current stack pointer
(gdb) x/20wx $rsp
```

## Analyzing Stack Frames

```bash
# Show current stack frame
(gdb) info frame

# Show all stack frames (call stack)
(gdb) backtrace
(gdb) bt

# Move up/down stack frames
(gdb) up
(gdb) down

# Show local variables in current frame
(gdb) info locals

# Show function arguments
(gdb) info args
```

## Finding Buffer Overflows

The most common security debugging task: finding where input overflows a buffer.

### Example: Testing an Overflow Condition

```bash
# Generate a cyclic pattern to find the overflow offset
# Using python to create a De Bruijn sequence
python3 -c "
import string
chars = string.ascii_letters + string.digits
pattern = ''
for i in range(len(chars)):
    for j in range(len(chars)):
        pattern += chars[i] + chars[j]
print(pattern[:200])
" > /tmp/pattern.txt

# Run the binary with the pattern
(gdb) run $(cat /tmp/pattern.txt)

# When it crashes, check where RIP/EIP points
(gdb) info registers rip
# The value in RIP tells you the offset within the pattern
```

With pwndbg or PEDA, this is easier:

```bash
# In pwndbg
(gdb) cyclic 200
# Copy the output and run:
(gdb) run AAAaBAAcBAAd...

# On crash:
(gdb) cyclic -l $rsp    # Find offset automatically
```

### Inspecting a Stack Frame During a Crash

```bash
(gdb) run $(python3 -c "print('A'*200)")

# On segfault:
(gdb) info registers

# Typical output showing overwritten RIP:
# rip  0x4141414141414141  <-- 'AAAA...' has overwritten return address

# Show what's on the stack around the overflow
(gdb) x/40wx $rsp-80
```

## Examining Function Calls

```bash
# Disassemble a function
(gdb) disassemble main
(gdb) disassemble /m main    # Mix source and assembly if symbols available

# Set a breakpoint before a dangerous function call
(gdb) break strcpy
(gdb) run test_input

# When stopped at strcpy, examine arguments
(gdb) info args
# Or manually: first arg in rdi, second in rsi (x86_64 calling convention)
(gdb) x/s $rdi    # destination buffer
(gdb) x/s $rsi    # source string
(gdb) print (int)strlen($rsi)    # length of input
```

## Watchpoints: Monitoring Memory Changes

```bash
# Watch for writes to a specific memory address
(gdb) watch *0x00404010

# Watch for reads
(gdb) rwatch *0x00404010

# Watch a variable (if symbols available)
(gdb) watch password_buf
```

When the watched location changes, GDB pauses and shows you what caused the change — invaluable for tracking how attacker-controlled input propagates through memory.

## Scripting GDB with Python

Automate repetitive analysis:

```python
# gdb_audit.py — run with: gdb -x gdb_audit.py ./target

import gdb

class SecurityAudit(gdb.Command):
    def __init__(self):
        super(SecurityAudit, self).__init__("audit", gdb.COMMAND_USER)

    def invoke(self, arg, from_tty):
        # Run to main
        gdb.execute("break main")
        gdb.execute("run")

        # Check for dangerous functions
        dangerous_funcs = ['strcpy', 'gets', 'sprintf', 'system', 'exec']
        print("\n[*] Checking for dangerous function calls:")
        for fn in dangerous_funcs:
            result = gdb.execute(f"info functions {fn}", to_string=True)
            if fn in result:
                print(f"  [!] Found: {fn}")

        # Show security mitigations
        print("\n[*] Binary security properties:")
        gdb.execute("checksec")

SecurityAudit()
```

```bash
gdb -x gdb_audit.py ./target
(gdb) audit
```

## Remote Debugging for Embedded Targets

```bash
# On the target device (ARM, MIPS, etc.)
gdbserver 0.0.0.0:1234 ./target_binary

# On your analysis machine
gdb-multiarch ./target_binary
(gdb) set architecture arm
(gdb) target remote 192.168.1.100:1234
(gdb) continue
```

This is essential for analyzing firmware running on embedded systems where installing GDB locally is impractical.

## Quick Security Triage Commands

When you get an unknown binary, run these in sequence:

```bash
gdb ./unknown_binary

(gdb) info file        # File type and architecture
(gdb) info functions   # List all function symbols
(gdb) checksec         # Security mitigations (PEDA/pwndbg)
(gdb) break main
(gdb) run
(gdb) info registers   # Initial register state
(gdb) x/20i $rip       # First 20 instructions
(gdb) bt               # Backtrace
```

## Related Reading

- [How to Use radare2 for Binary Analysis](/radare2-binary-analysis-guide/)
- [How to Use Ghidra for Reverse Engineering](/ghidra-reverse-engineering-guide/)
- [How to Use strace for Security Analysis](/strace-security-analysis-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
