---
layout: default
title: "How to Document Privacy Violations for Potential Class."
description: "A practical guide for developers and power users on collecting and preserving digital evidence of privacy violations for potential class action lawsuits."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-document-privacy-violations-for-potential-class-actio/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
---

{% raw %}
Documenting privacy violations requires preserving timestamped screenshots, browser inspector captures showing data transmission, network logs (using Charles or Fiddler), and metadata proving your access date and conditions, with parallel documentation of privacy policy text and company statements contemporaneous to the violation. Hash file timestamps, save browser consoles showing data being transmitted to unexpected third parties, and preserve ToS or privacy policy versions via archive.org before they're updated—this layer of authentication evidence transforms anecdotal complaints into admissible documentation. Class action lawyers specifically need evidence showing: (1) what data was actually collected, (2) violation of stated privacy policy or law, (3) company knowledge (security researcher disclosures, regulatory complaints), and (4) affected population scale—technical documentation you can provide demonstrating large-scale unencrypted data transmission, hardcoded API keys in client code, or retention beyond stated periods becomes crucial evidence distinguishing frivolous from meritorious class actions.

## Understanding Evidence Requirements

For privacy violations to support a class action, evidence must be **authentic**, **reliable**, and **preserved in forensically sound ways**. Courts look for documentation that shows:
- What data was collected
- How it was transmitted or stored
- Who had access to it
- When the violations occurred

Screen captures alone rarely suffice. You need reproducible, timestamped evidence that can be independently verified.

## Capture Network Traffic

One of the most compelling forms of evidence comes from observing actual data transmissions. Developers can use tools like Wireshark or mitmproxy to capture and analyze network traffic.

### Using mitmproxy for HTTPS Inspection

```bash
# Install mitmproxy
pip install mitmproxy

# Run mitmproxy in regular mode
mitmproxy -p 8080

# Configure your system or browser to route traffic through 127.0.0.1:8080
# Examine the traffic for unexpected data exfiltration
```

After capturing traffic, export the data:
```bash
mitmproxy -n -r capture_file.mitm --set flow_detail=3 > network_analysis.txt
```

This creates a permanent record showing exactly what data leaves your device and where it goes.

## Document API Calls and Data Leaks

For web applications, browser developer tools provide immediate evidence of privacy issues. Documenting API calls reveals what information gets sent to external servers.

### Capturing Console Logs

Open Chrome DevTools (F12), navigate to the Network tab, and record XHR/Fetch requests. Export these as HAR files:

1. Right-click in the Network tab
2. Select "Save all as HAR with content"
3. Save the .har file with a descriptive name

HAR files contain complete request/response cycles including headers and payloads.

### Automated API Monitoring Script

```python
#!/usr/bin/env python3
"""
Monitor outgoing HTTP requests and log them with timestamps.
"""
import json
import subprocess
from datetime import datetime
from pathlib import Path

def log_request(method, url, headers, body):
    entry = {
        "timestamp": datetime.utcnow().isoformat(),
        "method": method,
        "url": url,
        "headers": dict(headers),
        "body": body
    }
    
    log_file = Path("privacy_log.jsonl")
    with open(log_file, "a") as f:
        f.write(json.dumps(entry) + "\n")
    
    print(f"[{entry['timestamp']}] {method} {url}")

# Example using mitmproxy's API
# from mitmproxy import http
# def request(flow: http.HTTPFlow):
#     log_request(flow.request.method, str(flow.request.pretty_url),
#                 flow.request.headers, flow.request.get_text(strict=False))
```

## Screenshot Documentation with Metadata

Screenshots provide visual evidence but require careful handling. Always capture full-page screenshots that include URL bars and timestamps.

```bash
# Using Chrome headless for automated captures
chrome --headless --screenshot=evidence_$(date +%Y%m%d_%H%M%S).png \
  --virtual-time-budget=5000 \
  "https://example.com"
```

For timestamp verification, use services that embed metadata or capture browser console output alongside screenshots.

## Preserving Web Archives

Internet Archive's Wayback Machine provides immutable snapshots. Submit suspicious pages before they change:

```bash
# Use the Wayback Machine save API
curl -I "https://web.archive.org/web/$(date +%Y%m%d%H%M%S)/https://example.com/suspicious-page"
```

Create personal backups using wget:

```bash
wget --mirror --convert-links --adjust-extension \
  --page-requisites --no-parent \
  -e robots=off --timestamping \
  "https://example.com" \
  2>&1 | tee wget_$(date +%Y%m%d).log
```

The log file becomes part of your evidence chain, showing exactly when content was retrieved.

## Cookie and Storage Analysis

Privacy violations often involve unauthorized tracking. Document cookies and local storage:

### Extracting Browser Cookies

```javascript
// Run in browser console to export all cookies
const cookies = document.cookie;
const blob = new Blob([cookies], { type: 'text/plain' });
const url = URL.createObjectURL(blob);
const a = document.createElement('a');
a.href = url;
a.download = 'cookies_' + new Date().toISOString() + '.txt';
a.click();
```

For Firefox, use the sqlite3 command-line tool:

```bash
sqlite3 cookies.sqlite "SELECT host, name, value, expiry FROM moz_cookies" \
  > cookies_export.txt
```

## Building a Comprehensive Evidence Package

Organize collected evidence systematically:

```
evidence/
├── network_captures/
│   └── mitmproxy_dump.jsonl
├── screenshots/
│   └── evidence_*.png
├── web_archives/
│   └── example.com/
├── api_logs/
│   └── privacy_log.jsonl
└── metadata.json
```

Create a manifest file documenting each piece:

```json
{
  "case_id": "privacy_001",
  "collected": "2026-03-16T10:30:00Z",
  "collector": "theluckystrike",
  "evidence": [
    {
      "file": "network_captures/mitmproxy_dump.jsonl",
      "description": "HTTPS traffic capture showing data exfiltration",
      "tool": "mitmproxy 10.0.0"
    },
    {
      "file": "screenshots/evidence_20260316.png",
      "description": "Screenshot of suspicious API call",
      "tool": "Chrome headless"
    }
  ]
}
```

## Legal Considerations

Preserve evidence properly from the start. Chain of custody matters:
- Use write-once storage when possible
- Hash all files and record checksums
- Document who collected what and when
- Avoid modifying original files

Consult with an attorney before initiating any formal action. This guide provides technical documentation methods, not legal advice.

## Conclusion

Documenting privacy violations requires combining multiple evidence types: network captures, API logs, screenshots, and web archives. Developers and power users can leverage technical tools to create robust evidence packages that withstand legal scrutiny.

The key is consistency—document everything, maintain hash verification, and organize your findings systematically. When privacy violations affect thousands of users, your careful evidence collection could support meaningful accountability.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
