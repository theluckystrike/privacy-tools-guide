---
layout: default
title: "Local-Only Security Camera Setup Without Cloud Using Frigate"
description: "A practical guide for setting up Frigate NVR for completely local security camera monitoring without cloud dependencies. Keep your video footage private"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Privacy Tools Guide"
permalink: /local-only-security-camera-setup-without-cloud-using-frigate/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide, security]
---

{% raw %}

Deploy Frigate on a Docker-capable computer (mini-PC, Intel NUC, or Raspberry Pi) to record IP camera streams directly to local storage with no cloud upload, no subscriptions, and no monthly fees. Frigate performs real-time object detection (people, vehicles, animals) locally and integrates with Home Assistant for automation. Set up is straightforward: install Docker, configure your IP cameras' RTSP streams in Frigate's config file, and video records directly to attached storage. Your footage stays fully under your control.

## Table of Contents

- [Why Frigate for Local-Only Security](#why-frigate-for-local-only-security)
- [Hardware Requirements](#hardware-requirements)
- [Prerequisites](#prerequisites)
- [Performance Optimization](#performance-optimization)
- [Troubleshooting](#troubleshooting)

## Why Frigate for Local-Only Security

Frigate is a Docker-based NVR built specifically for Home Assistant but capable of standalone operation. It processes video streams locally using real-time object detection, distinguishing between people, vehicles, animals, and other motion. Unlike cloud services that upload your footage to remote servers, Frigate keeps everything on your network.

The advantages of this approach include zero subscription costs, complete data privacy since footage never leaves your property, reliable operation during internet outages, and minimal latency for real-time monitoring. You also avoid vendor lock-in and can export or delete footage whenever you want.

## Hardware Requirements

For a basic Frigate setup, you'll need:

- **A computer or single-board computer** capable of running Docker (Intel NUC, mini-PC, or Raspberry Pi 5 with external storage)
- **IP cameras** that support RTSP streams (Reolink, Amcrest, and many other brands work well)
- **Network switch** (preferably gigabit) to connect cameras and the NVR
- **Storage** for footage (external SSD or HDD recommended, at least 500GB for several days of recordings)

For the NVR itself, a mid-range mini-PC with an Intel processor provides hardware acceleration for video decoding, allowing handling of multiple camera streams simultaneously.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Install Frigate

The easiest way to run Frigate is using Docker Compose. Create a directory for Frigate and add the following to your `docker-compose.yml`:

```yaml
version: '3.8'
services:
  frigate:
    container_name: frigate
    image: ghcr.io/blakeblackshear/frigate:latest
    shm_size: '256mb'
    volumes:
      - ./config:/config
      - ./media:/media/frigate
      - /etc/localtime:/etc/localtime:ro
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "5000:5000"
      - "8554:8554"
      - "8555:8555/tls"
    environment:
      FRIGATE_RTSP_PASSWORD: "your_secure_password"
```

Replace `your_secure_password` with a strong password. Start Frigate with:

```bash
docker compose up -d
```

Access the Frigate web interface at `http://your-server-ip:5000`.

### Step 2: Configure Your Cameras

In the Frigate configuration file (`config/config.yml`), define your cameras. Here's an example for two cameras:

```yaml
mqtt:
  host: your_mqtt_broker_ip
  user: mqtt_user
  password: mqtt_password

cameras:
  front_door:
    enabled: true
    ffmpeg:
      inputs:
        - path: rtsp://admin:password@camera-ip:554/stream1
          roles:
            - detect
            - record
            - rtmp
    detect:
      width: 1280
      height: 720
      fps: 5
    record:
      enabled: true
      retain:
        days: 7
    snapshots:
      enabled: true
      retain:
        default: 7

  backyard:
    enabled: true
    ffmpeg:
      inputs:
        - path: rtsp://admin:password@camera-ip-2:554/stream1
          roles:
            - detect
            - record
    detect:
      width: 1920
      height: 1080
      fps: 5
    record:
      enabled: true
      retain:
        days: 3
```

Replace the camera IP addresses and credentials with your actual camera information. The RTSP path varies by camera manufacturer—consult your camera's documentation for the correct format.

### Step 3: Set Up Object Detection

Frigate uses OpenCV and TensorFlow for local object detection. The default configuration detects people, cats, dogs, and cars. Add detection zones to reduce false alerts:

```yaml
cameras:
  front_door:
    # ... previous camera config ...
    objects:
      track:
        - person
        - cat
        - dog
      filters:
        person:
          min_area: 5000
          max_area: 100000
          threshold: 0.7
```

This configuration focuses on person detection while filtering out small movements or distant objects that might trigger unnecessary recordings.

### Step 4: Integrate with Home Assistant

For integration, add the Frigate integration in Home Assistant by adding this to your `configuration.yaml`:

```yaml
frigate:
  url: "http://your-frigate-ip:5000"
  camera_ui_enabled: true
```

After restarting Home Assistant, you'll have access to Frigate camera entities, motion sensors, and can create automations based on detected objects.

### Step 5: Storage and Retention

Configure retention policies based on your storage capacity and needs:

```yaml
record:
  enabled: true
  retain:
    days: 7
    mode: motion

snapshots:
  enabled: true
  retain:
    default: 7
    mode: motion
```

Consider setting up a separate storage mount for long-term archives if you need to retain footage beyond the default retention period.

### Step 6: Remote Access Without Cloud

To access your cameras remotely without cloud services, set up a VPN connection to your home network. WireGuard provides a lightweight, secure option that works well for remote camera viewing. Install WireGuard on your phone and home router, then connect to your network when you need to view live footage or review recordings.

Alternatively, Home Assistant's remote access through Nabu Casa works without opening ports on your router, though this does require a subscription. For completely local operation, WireGuard or a similar VPN solution is recommended.

## Performance Optimization

If you experience performance issues with multiple cameras, consider these adjustments:

- Enable hardware acceleration by adding `VAAPI` or `QSV` to your Docker runtime
- Reduce detection resolution for cameras you don't monitor closely
- Limit the number of detection zones and object types
- Use motion-based recording instead of continuous recording to save resources

### Step 7: Camera Selection Guide

Not all cameras work equally well with Frigate:

| Camera Feature | Why It Matters | Recommended |
|---------------|---------------|-------------|
| RTSP support | Required for Frigate integration | Reolink, Amcrest, Hikvision |
| ONVIF compliance | Standardized camera control | Most commercial IP cameras |
| PoE (Power over Ethernet) | Single cable for power and data | Reolink RLC-810A, Amcrest IP8M |
| Local storage | SD card backup if NVR fails | Most Reolink models |
| No cloud requirement | Camera works without internet | Reolink, Amcrest (avoid Ring, Nest) |

Avoid cameras that require cloud connectivity. Ring, Nest, and Arlo depend on cloud services and may stop working without internet access.

### Step 8: Network Isolation for Camera Security

Place your cameras on a separate VLAN to prevent them from reaching the internet:

```bash
# Firewall rule: block camera VLAN from internet
iptables -A FORWARD -i br-cameras -o eth0 -j DROP
# Allow cameras to reach Frigate only
iptables -A FORWARD -i br-cameras -d 192.168.1.100 -p tcp --dport 5000 -j ACCEPT
```

This prevents compromised cameras from phoning home or being used as entry points into your network.

### Step 9: Backup Strategy for Recorded Footage

Configure automated backups of critical footage:

```bash
#!/bin/bash
# backup-footage.sh
FRIGATE_MEDIA="/path/to/frigate/media"
BACKUP_DRIVE="/mnt/backup/cameras"

rsync -av --include='*/' --include='*.mp4' --exclude='*' \
  "$FRIGATE_MEDIA/recordings/" "$BACKUP_DRIVE/recordings/"

# Clean backups older than 30 days
find "$BACKUP_DRIVE" -name "*.mp4" -mtime +30 -delete
```

Run this script nightly to maintain an independent backup without duplicating continuous footage.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to without cloud using frigate?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [How to Set up Local Network Storage for Security](/privacy-tools-guide/how-to-set-up-local-network-storage-for-security-cameras-without-nas-cloud/)
- [Cloud Storage Security Breach History: Compromised](/privacy-tools-guide/cloud-storage-security-breach-history-compromised-services-t/)
- [Air Gapped Computer Setup For Maximum Security Practical](/privacy-tools-guide/air-gapped-computer-setup-for-maximum-security-practical-gui/)
- [Best Hardware Security Key Comparison: A Developer's Guide](/privacy-tools-guide/best-hardware-security-key-comparison/)
- [Eufy Camera Cloud Upload Controversy What Local Storage](/privacy-tools-guide/eufy-camera-cloud-upload-controversy-what-local-storage/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
