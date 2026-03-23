---
layout: default

permalink: /privacy-focused-website-speed-test-tool-that-does-not-track-/
description: "Learn privacy focused website speed test tool that does not track  with practical examples, tips, and step-by-step instructions for getting the best results."
tags: [privacy-tools-guide, privacy]
author: "Privacy Tools Guide"
reviewed: true
score: 9
date: 2026-03-15
categories: [guides]
---

layout: default
title: "Privacy-Focused Website Speed Test Tool That Does Not Track"
description: "Website speed test tools that never log tested URLs. Self-hosted options, CLI tools, and offline benchmarks for privacy-conscious developers."
date: 2026-03-16
author: "Privacy Tools Guide"
permalink: /privacy-focused-website-speed-test-tool-that-does-not-track-/
categories: [guides]
voice-checked: true
tags: [privacy-tools-guide]
reviewed: true
score: 8
intent-checked: true
---

{% raw %}

Most mainstream website speed test tools collect and store the URLs you test, often sharing them with third parties or using them for analytics. For developers and power users working with sensitive projects, client websites, or confidential web applications, this data collection poses real privacy risks. This guide covers privacy-focused alternatives that measure website performance without tracking or logging tested URLs.

## Table of Contents

- [Why URL Tracking Matters in Speed Testing](#why-url-tracking-matters-in-speed-testing)
- [Self-Hosted Speed Testing with WebPageTest](#self-hosted-speed-testing-with-webpagetest)
- [Command-Line Speed Testing Tools](#command-line-speed-testing-tools)
- [Building a Custom Speed Test Dashboard](#building-a-custom-speed-test-dashboard)
- [Comparing Public Speed Test Privacy Policies](#comparing-public-speed-test-privacy-policies)
- [Integrating Speed Testing Into CI/CD Pipelines](#integrating-speed-testing-into-cicd-pipelines)
- [Privacy Verification Tips](#privacy-verification-tips)
- [Conclusion](#conclusion)
- [Advanced Metrics: Beyond Basic Speed](#advanced-metrics-beyond-basic-speed)
- [Threat Model: What URL Testing Leaks](#threat-model-what-url-testing-leaks)
- [Building a Monitoring Dashboard](#building-a-monitoring-dashboard)
- [Continuous Integration: CI/CD Speed Testing](#continuous-integration-cicd-speed-testing)

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

### Chrome DevTools Protocol for Analysis

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

## Advanced Metrics: Beyond Basic Speed

Modern web performance includes metrics that simple curl tests miss. Implement measurement:

```javascript
// comprehensive-metrics.js - Full page performance analysis
const puppeteer = require('puppeteer');
const fs = require('fs');

async function measureComprehensively(url) {
  const browser = await puppeteer.launch({
    headless: 'new',
    args: ['--no-sandbox']
  });

  const page = await browser.newPage();

  // Clear cache for consistent measurements
  const context = browser.defaultBrowserContext();
  await context.clearCookies();

  // Capture all metrics
  const metrics = {
    url: url,
    timestamp: new Date().toISOString(),
    performance: {},
    resources: {
      js: 0,
      css: 0,
      images: 0,
      other: 0
    }
  };

  // Performance API metrics
  const perfData = await page.evaluate(() => {
    const perf = window.performance;
    return {
      navigationStart: perf.timing.navigationStart,
      domLoaded: perf.timing.domContentLoadedEventEnd,
      pageLoaded: perf.timing.loadEventEnd,
      firstPaint: perf.getEntriesByType('paint'),
      largestContentfulPaint: perf.getEntriesByType('largest-contentful-paint'),
      layoutShift: perf.getEntriesByType('layout-shift')
    };
  });

  // Measure actual resource loading
  const resourceMetrics = await page.evaluate(() => {
    return performance.getEntriesByType('resource').map(r => ({
      name: r.name,
      type: r.initiatorType,
      duration: r.duration,
      size: r.transferSize
    }));
  });

  // Network conditions simulation
  const conditions = [
    { name: '4G', download: 4000, upload: 3000, latency: 50 },
    { name: '3G', download: 1600, upload: 750, latency: 100 },
    { name: 'Slow 4G', download: 400, upload: 400, latency: 200 }
  ];

  for (const condition of conditions) {
    await page.emulateNetworkConditions(
      true,
      condition.download,
      condition.upload,
      condition.latency
    );

    const loadTime = await page.goto(url, { waitUntil: 'networkidle2' });
    metrics.performance[condition.name] = {
      status: loadTime.status(),
      loadMs: (new Date() - new Date(0)).valueOf() // Simplified
    };
  }

  // Analyze resources
  for (const resource of resourceMetrics) {
    const ext = resource.type.toLowerCase();
    if (ext.includes('script')) metrics.resources.js++;
    else if (ext.includes('stylesheet')) metrics.resources.css++;
    else if (ext.includes('image')) metrics.resources.images++;
    else metrics.resources.other++;
  }

  await browser.close();
  return metrics;
}

// Run and store results locally
(async () => {
  const testUrl = process.argv[2] || 'https://example.com';
  const results = await measureComprehensively(testUrl);

  // Save to local file, never transmit
  const resultsDir = `${process.env.HOME}/.local/share/speed-tests`;
  fs.mkdirSync(resultsDir, { recursive: true });

  const filename = `${resultsDir}/test_${Date.now()}.json`;
  fs.writeFileSync(filename, JSON.stringify(results, null, 2));

  console.log(`Results saved to ${filename}`);
  console.log(JSON.stringify(results, null, 2));
})();
```

## Threat Model: What URL Testing Leaks

Testing a URL externally leaks more than you might realize:

| Data Leaked | Risk | Mitigation |
|---|---|---|
| URL itself | Critical | Self-hosted or local test |
| Site structure | Medium | Testing through same ISP/VPN |
| Technology stack | Medium | Site fingerprinting possible |
| Geographic origin | Low | Public knowledge often |
| Update frequency | Medium | Pattern analysis possible |
| Authentication status | High | Could reveal access levels |
| File upload progress | High | Metadata about operations |

Self-hosting eliminates all of these risks by keeping the URL completely private.

## Building a Monitoring Dashboard

For ongoing performance tracking without cloud dependency:

```python
#!/usr/bin/env python3
# web-perf-dashboard.py

import subprocess
import json
import time
from pathlib import Path
from datetime import datetime, timedelta

class LocalPerfDashboard:
    def __init__(self, sites_config):
        self.sites = sites_config
        self.results_dir = Path.home() / '.local/share/web-perf'
        self.results_dir.mkdir(parents=True, exist_ok=True)

    def test_site(self, site_url):
        """Test single site using Chrome DevTools Protocol"""
        cmd = [
            'lighthouse',
            site_url,
            '--only-categories=performance',
            '--output=json',
            '--chrome-flags="--headless"',
            '--output-path=-'  # stdout
        ]

        try:
            result = subprocess.run(cmd, capture_output=True, text=True, timeout=120)
            data = json.loads(result.stdout)

            return {
                'url': site_url,
                'timestamp': datetime.now().isoformat(),
                'score': data['lighthouseResult']['categories']['performance']['score'] * 100,
                'first_contentful_paint': data['lighthouseResult']['audits']['first-contentful-paint']['numericValue'],
                'largest_contentful_paint': data['lighthouseResult']['audits']['largest-contentful-paint']['numericValue'],
            }
        except Exception as e:
            return {'url': site_url, 'error': str(e)}

    def run_daily_tests(self):
        """Run tests for all configured sites"""
        results = []

        for site_config in self.sites:
            print(f"Testing {site_config['name']}...")
            result = self.test_site(site_config['url'])
            results.append(result)

            # Store individual result
            filename = self.results_dir / f"{site_config['name']}_{datetime.now().isoformat()}.json"
            filename.write_text(json.dumps(result, indent=2))

        return results

    def generate_report(self, days=7):
        """Generate performance report for last N days"""
        cutoff = datetime.now() - timedelta(days=days)
        results_by_site = {}

        # Aggregate results
        for result_file in self.results_dir.glob('*.json'):
            if result_file.stat().st_mtime > cutoff.timestamp():
                data = json.loads(result_file.read_text())
                site = data['url']

                if site not in results_by_site:
                    results_by_site[site] = []

                results_by_site[site].append(data)

        # Calculate trends
        report = {}
        for site, measurements in results_by_site.items():
            scores = [m['score'] for m in measurements if 'score' in m]
            if scores:
                report[site] = {
                    'avg_score': sum(scores) / len(scores),
                    'min_score': min(scores),
                    'max_score': max(scores),
                    'measurements': len(scores)
                }

        return report

# Usage
sites_to_monitor = [
    {'name': 'main-site', 'url': 'https://example.com'},
    {'name': 'api-site', 'url': 'https://api.example.com'},
]

dashboard = LocalPerfDashboard(sites_to_monitor)
dashboard.run_daily_tests()

# Generate weekly report
weekly = dashboard.generate_report(days=7)
print(json.dumps(weekly, indent=2))
```

## Continuous Integration: CI/CD Speed Testing

Integrate privacy-preserving speed tests into your CI/CD pipeline:

```yaml
# .github/workflows/speed-test.yml
name: Local Speed Test

on:
  schedule:
    - cron: '0 9 * * 1'  # Weekly Monday 9 AM
  workflow_dispatch:

jobs:
  speed-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Install dependencies
        run: |
          npm install -g lighthouse puppeteer
          apt-get update && apt-get install -y chromium

      - name: Run speed tests
        run: |
          mkdir -p results
          lighthouse https://your-site.com --output=json --output-path=results/test.json

      - name: Archive results
        uses: actions/upload-artifact@v3
        with:
          name: speed-test-results
          path: results/

      - name: Commit results
        run: |
          git config user.name "Speed Test Bot"
          git config user.email "bot@example.com"
          git add results/
          git commit -m "Speed test: $(date)" || true
          git push
```

This approach keeps all testing data in your repository, never sending URLs to external services.

## Related Articles

- [Privacy-Focused Network Speed Test Comparison Tools That](/privacy-focused-network-speed-test-comparison-tools-that-res/)
- [Proton VPN vs Mullvad Speed Test and Privacy Audit 2026](/proton-vpn-vs-mullvad-speed-test-privacy-audit-2026/)
- [How to Optimize LibreWolf Browser Speed and Compatibility](/how-to-optimize-librewolf-browser-speed-and-compatibility-wi/)
- [How to Test if Your Anti-Fingerprinting Setup Actually](/how-to-test-if-your-anti-fingerprinting-setup-actually-works/)
- [Best Privacy-Focused Monitoring Tool That Does Not Collect](/best-privacy-focused-monitoring-tool-that-does-not-collect-s/)
- [Cursor Pro Privacy Mode Does It Cost Extra](https://bestremotetools.com/cursor-pro-privacy-mode-does-it-cost-extra-for-zero-retention/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
