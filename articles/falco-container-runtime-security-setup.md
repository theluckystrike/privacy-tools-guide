---
layout: default
title: "How to Set Up Falco for Container Runtime Security"
description: "Deploy Falco to detect suspicious container behavior in real time: kernel syscall monitoring, custom rules, alerting integrations, and Kubernetes deployment"
date: 2026-03-22
author: theluckystrike
permalink: /falco-container-runtime-security-setup/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, security]
---

{% raw %}

# How to Set Up Falco for Container Runtime Security

Falco is a CNCF-graduated runtime security tool that monitors Linux system calls to detect anomalous behavior in containers and Kubernetes workloads. It catches things static scanners miss: a container spawning a shell, a process reading credentials files, unexpected network connections from a web server. This guide deploys and configures Falco from scratch.

## How Falco Works

Falco intercepts kernel system calls using either:

- **Kernel module** — compiled and loaded into the kernel (traditional approach)
- **eBPF probe** — safer, no kernel module required (recommended for production)

Every syscall is evaluated against a rule set. When a rule matches, Falco emits an alert to stdout, a log file, syslog, or a webhook.

## Prerequisites

- Linux kernel 4.14+ (5.x recommended for eBPF)
- Docker or Kubernetes cluster
- Root access

## Step 1 — Install Falco

```bash
# Add Falco repository
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] \
  https://download.falco.org/packages/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/falcosecurity.list

sudo apt update && sudo apt install -y falco
```

During installation, choose **eBPF** when prompted (safer than kernel module for modern systems).

## Step 2 — Verify Installation

```bash
sudo systemctl start falco
sudo systemctl status falco

# Check that Falco is generating events
sudo journalctl -u falco -f
```

Trigger a test alert:

```bash
# Falco watches for shells spawned in containers
# Run a container and spawn a shell — Falco should alert
docker run --rm -it alpine sh -c "cat /etc/shadow"

# Check for the alert
sudo journalctl -u falco --since "1 minute ago" | grep -i shadow
```

## Step 3 — Understand the Default Rules

Falco ships with a default ruleset at `/etc/falco/falco_rules.yaml`. Review key rules:

```bash
# View all rule names
grep "^- rule:" /etc/falco/falco_rules.yaml
```

Key defaults worth knowing:

- **Terminal shell in container** — any interactive shell spawned in a container
- **Write below binary dir** — file write to `/bin`, `/usr/bin`, etc.
- **Read sensitive file untrusted** — access to `/etc/shadow`, `/etc/passwd`, SSH keys
- **Outbound connection to C2 server** — connections to known threat intelligence IPs
- **Unexpected network connection** — process connecting out from a container unexpectedly
- **Container run as root** — container processes running as UID 0

## Step 4 — Write Custom Rules

Create `/etc/falco/falco_rules.local.yaml` for your custom rules (never modify the default file):

```yaml
# /etc/falco/falco_rules.local.yaml

# Alert when any process in a web server container touches /etc/
- rule: Web Container Accessing /etc
  desc: A web server container is accessing sensitive /etc files
  condition: >
    container and
    container.image.repository contains "nginx" and
    (open_read or open_write) and
    fd.directory startswith "/etc"
  output: >
    Web container accessed /etc (user=%user.name
    container=%container.name image=%container.image.repository
    file=%fd.name proc=%proc.name)
  priority: WARNING

# Alert on curl/wget in production containers
- rule: Unexpected Download Tool in Container
  desc: curl or wget running in a container that shouldn't need them
  condition: >
    container and
    proc.name in (curl, wget, fetch) and
    not container.image.repository in (my-trusted-image)
  output: >
    Download tool executed in container
    (user=%user.name container=%container.name
    image=%container.image.repository cmdline=%proc.cmdline)
  priority: WARNING

# Alert on new user creation
- rule: User Added to Sudoers
  desc: A user was added to sudoers inside a container
  condition: >
    container and
    (open_write) and
    fd.name in (/etc/sudoers, /etc/sudoers.d)
  output: >
    Sudoers file modified (user=%user.name
    container=%container.name file=%fd.name)
  priority: CRITICAL
```

Reload Falco rules without restart:

```bash
sudo kill -1 $(pidof falco)
```

## Step 5 — Configure Alert Outputs

Edit `/etc/falco/falco.yaml`:

```yaml
# stdout and file output
stdout_output:
  enabled: true

file_output:
  enabled: true
  keep_alive: false
  filename: /var/log/falco/events.log

# Syslog — useful for forwarding to Graylog or Splunk
syslog_output:
  enabled: true

# HTTP webhook — for Slack, PagerDuty, etc.
http_output:
  enabled: true
  url: https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
  user_agent: "falcosecurity/falco"
```

## Step 6 — Deploy Falco on Kubernetes

For Kubernetes, deploy Falco as a DaemonSet using Helm:

```bash
# Add Falco Helm repo
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

# Install Falco with eBPF driver
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true

# Verify pods are running
kubectl get pods -n falco
```

Falco Sidekick routes alerts to 50+ outputs including Slack, PagerDuty, Elasticsearch, and Datadog.

## Step 7 — Kubernetes-Specific Rules

Add K8s-aware rules to catch cluster-level threats:

```yaml
# Detect exec into a running pod (common attacker behavior)
- rule: Kubectl Exec Into Pod
  desc: kubectl exec was used to get into a running pod
  condition: >
    ka.verb = exec and
    ka.target.resource = pods
  output: >
    kubectl exec into pod (user=%ka.user.name
    pod=%ka.target.name ns=%ka.target.namespace)
  priority: WARNING
  source: k8s_audit

# Detect privilege escalation
- rule: Create Privileged Pod
  desc: A privileged container was created
  condition: >
    ka.verb = create and
    ka.target.resource = pods and
    ka.req.pod.containers.privileged = true
  output: >
    Privileged pod created (user=%ka.user.name
    pod=%ka.target.name ns=%ka.target.namespace)
  priority: CRITICAL
  source: k8s_audit
```

## Step 8 — Performance Tuning

Falco can generate high alert volume in busy environments. Reduce noise:

```yaml
# In falco.yaml — suppress frequent benign rules
base_syscalls:
  custom_set:
    - open
    - read
    - write
    # Remove syscalls not needed by your workloads

# Whitelist known-good containers from specific rules
# In falco_rules.local.yaml:
- rule: Terminal shell in container
  append: true
  condition: and not container.image.repository in (my-debug-image, my-admin-tool)
```

## Testing Your Rules

Falco provides a test suite:

```bash
# Install event-generator to test rules
docker run --rm falcosecurity/event-generator run syscall

# This generates test events that should trigger default rules
# Watch for alerts:
sudo journalctl -u falco -f
```

## Related Reading

- [How to Set Up Cilium for Network Security](/cilium-network-security-setup-guide/)
- [How to Set Up Graylog for Log Management](/graylog-log-management-setup-guide/)
- [Secure MQTT Broker Setup for IoT](/secure-mqtt-broker-setup-iot/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
