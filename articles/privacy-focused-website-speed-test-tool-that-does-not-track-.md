---
layout: default
title: "Privacy-Focused Website Speed Test Tool That Does Not Track Tested URLs"
description: "Discover privacy-focused website speed test tools that do not log or share tested URLs. This guide covers self-hosted options, CLI tools, and techniques for developers who need performance data without compromising URL privacy."
date: 2026-03-16
author: "Privacy Tools Guide"
permalink: /privacy-focused-website-speed-test-tool-that-does-not-track-/
categories: [guides]
voice-checked: true
tags: [privacy-tools-guide]
reviewed: true
score: 8
---

{% raw %}

Most mainstream website speed test tools collect and store the URLs you test, often sharing them with third parties or using them for analytics. For developers and power users working with sensitive projects, client websites, or confidential web applications, this data collection poses real privacy risks. This guide covers privacy-focused alternatives that measure website performance without tracking or logging tested URLs.

## Why URL Tracking Matters in Speed Testing

When you run a speed test on a client website or internal application through a public service, that URL enters someone else's database. Popular speed testing platforms may retain these records indefinitely, creating several concerns:

- **Client confidentiality**: Testing a client's staging site reveals its existence to a third party
- **Competitive intelligence**: Competitors could discover unreleased products or features
- **Security research**: Bug bounty researchers testing vulnerability disclosure programs may inadvertently expose program URLs
- **Compliance requirements**: Some industries require audit trails showing who accessed what systems

The solution involves using speed testing tools that either run entirely locally or operate under strict privacy policies with verifiable no-logging practices.

## Self-Hosted Speed Testing with WebPageTest

WebPageTest offers a self-hosted option that gives you complete control over data handling. Running your own instance means no URL data leaves your infrastructure.

### Setting Up a Private WebPageTest Instance

Deploy WebPageTest using Docker on your own server:

```bash
# Clone the WebPageTest server repository
git clone https://github.com/WPO-Foundation/webpagetest.git
cd webpagetest

# Start the server with Docker
docker run -d --name wptserver \
  -p 4000:80 \
  -p 4001:443 \
  -v $(pwd)/www:/var/www/html/results \
  -v $(pwd)/settings:/var/www/html/settings \
  webpagetest/server:latest
```

Configure your private instance by editing the settings file:

```ini
[settings]
title = "Private Speed Test"
contact = "your-team@company.com"

[privacy]
noarchive = 1
connect_secret = your_secret_token_here
```

Access your private speed test at `http://your-server:4000`. All test results remain on your infrastructure, and you can disable result archiving entirely through configuration.

## Command-Line Speed Testing Tools

For developers who prefer terminal-based workflows, several CLI tools provide privacy-focused speed testing without sending URLs to external services.

### Using curl for Basic Performance Measurements

The simplest approach uses standard Unix tools to measure response times:

```bash
# Measure time to first byte (TTFB)
curl -o /dev/null -s -w "TTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" https://example.com

# Test with different connection options
curl -o /dev/null -s -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\n" https://example.com
```

This method sends no data to external servers—the measurement happens entirely on your machine using your network connection to the target server.

### Chrome DevTools Protocol for Comprehensive Analysis

For more detailed performance data, use Puppeteer or Playwright with the Chrome DevTools Protocol:

```javascript
// performance-test.js - Run with Node.js
const puppeteer = require('puppeteer');

async function measurePerformance(url) {
  const browser = await puppeteer.launch({ headless: 'new' });
  const page = await browser.newPage();
  
  // Enable performance metrics
  await page.setCacheEnabled(false);
  
  const metrics = await page.metrics();
  const navigationTiming = await page.evaluate(() => {
    return JSON.stringify(performance.timing);
  });
  
  const timing = JSON.parse(navigationTiming);
  const loadTime = timing.loadEventEnd - timing.navigationStart;
  const domContent = timing.domContentLoadedEventEnd - timing.navigationStart;
  
  console.log(`URL: ${url}`);
  console.log(`Page Load Time: ${loadTime}ms`);
  console.log(`DOM Content Loaded: ${domContent}ms`);
  console.log(`JS Heap Size: ${metrics.JSHeapUsedSize / 1024 / 1024}MB`);
  
  await browser.close();
}

measurePerformance(process.argv[2] || 'https://example.com');
```

Run this script locally without any external dependencies:

```bash
node performance-test.js https://your-website.com
```

### Lighthouse CLI for Performance Audits

Google Lighthouse provides detailed performance analysis through its CLI tool:

```bash
# Install Lighthouse globally
npm install -g lighthouse

# Run a privacy-preserving audit (no external URLs stored)
lighthouse https://example.com \
  --only-categories=performance \
  --output=json \
  --output-path=./results.json \
  --chrome-flags="--headless"
```

Lighthouse sends requests from your local machine to the target site. The only data leaving your network is the actual HTTP requests to measure performance—no URL logging occurs.

## Building a Custom Speed Test Dashboard

For teams needing regular speed testing across multiple URLs, create a simple dashboard using Python:

```python
#!/usr/bin/env python3
# speed_test_dashboard.py

import time
import urllib.request
import json
from datetime import datetime

def test_url(url, runs=3):
    results = []
    
    for _ in range(runs):
        start = time.perf_counter()
        
        req = urllib.request.Request(url, headers={
            'User-Agent': 'PrivacySpeedTest/1.0'
        })
        
        with urllib.request.urlopen(req) as response:
            _ = response.read()
        
        elapsed = (time.perf_counter() - start) * 1000
        results.append(elapsed)
    
    return {
        'url': url,
        'avg_ms': sum(results) / len(results),
        'min_ms': min(results),
        'max_ms': max(results),
        'runs': runs,
        'tested_at': datetime.utcnow().isoformat()
    }

if __name__ == '__main__':
    urls = [
        'https://example.com',
        'https://your-client-site.com',
    ]
    
    for url in urls:
        result = test_url(url)
        print(f"{result['url']}: {result['avg_ms']:.2f}ms avg")
```

This script runs entirely locally with no external services involved. Extend it to store results in your own database or generate automated reports.

## Comparing Public Speed Test Privacy Policies

Not all public speed testing services are equally invasive. Understanding what different services actually log helps you make informed choices when self-hosting is not practical.

**GTmetrix** retains test results and associated URLs for up to 90 days by default on free accounts. Logged-in users can delete individual tests, but the data is retained server-side during that window. GTmetrix's parent company also uses aggregate data for benchmarking reports.

**Pingdom** stores test data including origin URLs and result sets. Their privacy policy permits sharing aggregated data with partners, though individual URLs are not publicly broadcast. Account-based usage links your identity to every tested URL.

**PageSpeed Insights (Google)** sends URLs to Google's infrastructure for testing. Google's privacy policy applies broadly, and URLs tested through PageSpeed Insights may inform Google's understanding of your web properties. For confidential projects, this is a meaningful concern.

**WebPageTest (public instance)** stores results publicly by default, including the full tested URL, waterfall charts, and connection details. Anyone with the result link can see what you tested. The private option requires an account and extends retention to 13 months.

By contrast, tools running entirely on your hardware — Lighthouse CLI, curl, Puppeteer scripts — leave no external footprint beyond the actual HTTP requests to the target server.

## Integrating Speed Testing Into CI/CD Pipelines

For development teams, integrating performance checks into automated pipelines provides continuous visibility without manual testing. This approach keeps all data internal to your infrastructure.

### GitHub Actions with Lighthouse

Run Lighthouse audits as part of your pull request workflow:

```yaml
name: Performance Check

on:
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Lighthouse
        run: npm install -g lighthouse

      - name: Run performance audit
        run: |
          lighthouse ${{ secrets.STAGING_URL }} \
            --only-categories=performance \
            --output=json \
            --output-path=./lighthouse-report.json \
            --chrome-flags="--headless --no-sandbox"

      - name: Parse performance score
        run: |
          SCORE=$(cat lighthouse-report.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(int(d['categories']['performance']['score']*100))")
          echo "Performance score: $SCORE"
          if [ "$SCORE" -lt 80 ]; then
            echo "Performance below threshold"
            exit 1
          fi
```

The staging URL comes from a repository secret, keeping it out of logs. Lighthouse runs locally within the GitHub Actions runner — no URLs are sent to external speed-testing services.

### Storing Historical Performance Data

Track performance trends over time with a simple SQLite store:

```python
#!/usr/bin/env python3
# perf_tracker.py

import sqlite3
import subprocess
import json
import sys
from datetime import datetime

DB_PATH = "/var/lib/perf_tracker/metrics.db"

def init_db(conn):
    conn.execute("""
        CREATE TABLE IF NOT EXISTS measurements (
            id INTEGER PRIMARY KEY,
            url TEXT NOT NULL,
            load_time_ms REAL,
            ttfb_ms REAL,
            score INTEGER,
            measured_at TEXT
        )
    """)
    conn.commit()

def record_measurement(conn, url, load_ms, ttfb_ms, score):
    conn.execute(
        "INSERT INTO measurements (url, load_time_ms, ttfb_ms, score, measured_at) VALUES (?, ?, ?, ?, ?)",
        (url, load_ms, ttfb_ms, score, datetime.utcnow().isoformat())
    )
    conn.commit()

def trend_report(conn, url, days=30):
    rows = conn.execute(
        """SELECT measured_at, load_time_ms, score FROM measurements
           WHERE url = ? AND measured_at > datetime('now', ?)
           ORDER BY measured_at ASC""",
        (url, f'-{days} days')
    ).fetchall()

    if not rows:
        print(f"No data for {url}")
        return

    avg_load = sum(r[1] for r in rows) / len(rows)
    avg_score = sum(r[2] for r in rows) / len(rows)
    print(f"URL: {url} | 30-day avg load: {avg_load:.0f}ms | avg score: {avg_score:.0f}")
```

This keeps all performance history on your own server with no external dependencies.

## Privacy Verification Tips

When evaluating any speed testing tool, verify its privacy claims:

1. **Check network traffic**: Use Wireshark or your browser's network inspector to confirm no unexpected requests go to analytics endpoints
2. **Review source code**: Open-source tools allow you to verify no URL exfiltration occurs
3. **Test with a unique URL**: Create a URL with a distinctive subdomain, then search for it in public databases to see if it was logged
4. **Read the privacy policy**: Look for explicit statements about URL retention and third-party sharing

## Conclusion

Privacy-focused website speed testing requires moving away from public services that log URLs. Self-hosted solutions like WebPageTest, CLI tools using curl or Puppeteer, and custom scripts all provide accurate performance data while keeping your tested URLs private. Choose the approach that matches your technical requirements and infrastructure capabilities.

For teams regularly testing multiple sites, investing in a self-hosted WebPageTest instance or building a custom dashboard pays dividends in both privacy and control over your testing workflow.

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
