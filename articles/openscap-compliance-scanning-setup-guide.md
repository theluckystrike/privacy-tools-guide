---
layout: default
title: "How to Set Up OpenSCAP for Compliance Scanning"
description: "Run CIS and STIG compliance scans on Linux with OpenSCAP, fix failed checks automatically with remediation scripts, and generate HTML audit reports"
date: 2026-03-22
author: theluckystrike
permalink: /openscap-compliance-scanning-setup-guide/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}
How to Set Up OpenSCAP for Compliance Scanning

OpenSCAP is the open-source implementation of the SCAP (Security Content Automation Protocol) standard. It scans your system against published security baselines. CIS Benchmarks, DISA STIGs, PCI DSS profiles. produces scored HTML reports, and generates Bash remediation scripts that automatically fix failures. This guide uses RHEL/CentOS 9 and Ubuntu 24.04.

What OpenSCAP Checks

- User and group account settings (UID 0 accounts, password aging)
- Service configuration (SSH, FTP, Telnet, NFS hardening)
- Filesystem mounts (noexec, nosuid, nodev on /tmp, /dev/shm)
- Kernel parameters (sysctl: IP forwarding, ICMP redirects, SYN cookies)
- Audit logging rules
- File permissions on critical system files
- Software update configuration
- SELinux/AppArmor status

---

1. Install OpenSCAP

```bash
RHEL / CentOS / Fedora
sudo dnf install -y openscap openscap-scanner openscap-utils scap-security-guide

Ubuntu / Debian
sudo apt install -y libopenscap8 openscap-scanner python3-openscap ssg-base ssg-debderived

Verify
oscap --version
```

---

2. List Available Profiles

The SCAP Security Guide (SSG) ships XML data stream files containing multiple profiles:

```bash
RHEL / CentOS
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

Ubuntu
oscap info /usr/share/xml/scap/ssg/content/ssg-ubuntu2404-ds.xml

Sample output:
Profiles:
    Title: CIS Red Hat Enterprise Linux 9 Benchmark for Level 1 - Server
        Id: xccdf_org.ssgproject.content_profile_cis_server_l1
    Title: DISA STIG for Red Hat Enterprise Linux 9
        Id: xccdf_org.ssgproject.content_profile_stig
```

---

3. Run Your First Scan

```bash
RHEL. CIS Level 1 Server profile
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --results /tmp/scan-results.xml \
  --report /tmp/scan-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

Ubuntu. CIS Level 1
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_level1_server \
  --results /tmp/scan-results.xml \
  --report /tmp/scan-report.html \
  /usr/share/xml/scap/ssg/content/ssg-ubuntu2404-ds.xml

echo "Report generated: /tmp/scan-report.html"
```

Open the HTML report in a browser. it shows pass/fail for each rule with explanations and references.

---

4. Read the Results

```bash
Quick summary from the XML results
oscap xccdf generate guide \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml \
  > /tmp/guide.html

Count passes and failures
grep -o '<result>.*</result>' /tmp/scan-results.xml | sort | uniq -c

List only failed rules
oscap xccdf results-arf /tmp/scan-results.xml 2>/dev/null || \
python3 - <<'EOF'
import xml.etree.ElementTree as ET
ns = {'xccdf': 'http://checklists.nist.gov/xccdf/1.2'}
tree = ET.parse('/tmp/scan-results.xml')
root = tree.getroot()
for result in root.iter('{http://checklists.nist.gov/xccdf/1.2}rule-result'):
    outcome = result.find('{http://checklists.nist.gov/xccdf/1.2}result')
    if outcome is not None and outcome.text == 'fail':
        print(result.attrib.get('idref', ''))
EOF
```

---

5. Generate and Apply Remediation Scripts

```bash
Generate a Bash script that fixes all failures
sudo oscap xccdf generate fix \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --result-id "" \
  --fix-type bash \
  --output /tmp/remediation.sh \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

REVIEW the script before running. some fixes restart services
less /tmp/remediation.sh

Run remediation (test on non-production first)
sudo bash /tmp/remediation.sh 2>&1 | tee /tmp/remediation.log

Re-scan after remediation to verify improvements
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --results /tmp/post-remediation-results.xml \
  --report /tmp/post-remediation-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

---

6. Ansible Playbook Remediation (Infrastructure as Code)

For automated fleet compliance:

```bash
Generate Ansible playbook instead of shell script
sudo oscap xccdf generate fix \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --fix-type ansible \
  --output /tmp/remediation-playbook.yml \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

Run on a remote host
ansible-playbook -i "targethost," -u root \
  /tmp/remediation-playbook.yml
```

---

7. Scan a Specific Rule (Targeted Checks)

```bash
List rule IDs
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml \
  | grep "Id:" | grep -v profile

Scan only the SSH hardening rules
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --rule xccdf_org.ssgproject.content_rule_sshd_disable_root_login \
  --rule xccdf_org.ssgproject.content_rule_sshd_set_max_sessions \
  --rule xccdf_org.ssgproject.content_rule_sshd_use_strong_macs \
  --results /tmp/ssh-results.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

---

8. Scan Docker Container Images

OpenSCAP can scan container images offline:

```bash
Install container scanning plugin
sudo dnf install -y openscap-containers   # RHEL

Pull and scan a container image
docker pull registry.access.redhat.com/ubi9/ubi
oscap-docker image-cve registry.access.redhat.com/ubi9/ubi \
  --report /tmp/container-report.html

Scan running container
oscap-docker container-cve <container_id> \
  --report /tmp/container-cve.html
```

---

9. DISA STIG Scan (Government/DoD Compliance)

```bash
DISA STIG for RHEL 9
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --results /tmp/stig-results.xml \
  --report /tmp/stig-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

Generate STIG Viewer compatible XCCDF results
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --results-arf /tmp/stig-arf.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

Import `/tmp/stig-arf.xml` into the DISA STIG Viewer tool for checklist generation.

---

10. Automate with Cron

```bash
Daily compliance scan, report emailed to security team
sudo tee /etc/cron.d/openscap > /dev/null <<'EOF'
0 2 * * * root /usr/bin/oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --results /var/log/oscap/results-$(date +\%Y\%m\%d).xml \
  --report /var/log/oscap/report-$(date +\%Y\%m\%d).html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml \
  && mail -s "OpenSCAP Report $(date +\%Y-\%m-\%d)" \
    -A /var/log/oscap/report-$(date +\%Y\%m\%d).html \
    security@example.com < /dev/null
EOF

sudo mkdir -p /var/log/oscap
```

---

Interpreting the Score

| Score Range | Posture |
|---|---|
| 90–100% | Strong compliance; investigate remaining failures |
| 70–89% | Moderate; address high-severity failures first |
| 50–69% | Weak; systematic hardening needed |
| Below 50% | Critical; remediate before production exposure |

Focus on severity=high failures first. Many low-severity failures are cosmetic or context-dependent.

---

Related Reading

- [How to Set Up Wazuh SIEM for Small Teams](/wazuh-siem-small-teams-setup-guide/)
- [How to Configure SELinux Policies Step by Step](/selinux-policy-configuration-guide/)
- [How to Set Up Snort IDS on Linux](/snort-ids-linux-setup-guide/)

---

Built by theluckystrike. More at [zovo.one](https://zovo.one)
{% endraw %}
