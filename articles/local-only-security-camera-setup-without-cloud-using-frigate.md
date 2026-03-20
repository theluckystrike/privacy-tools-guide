---
layout: default
title: "Local-Only Security Camera Setup Without Cloud Using Frigate"
description: "A practical guide for setting up Frigate NVR for completely local security camera monitoring without cloud dependencies. Keep your video footage private."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /local-only-security-camera-setup-without-cloud-using-frigate/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: false
---

{% raw %}

Deploy Frigate on a Docker-capable computer (mini-PC, Intel NUC, or Raspberry Pi) to record IP camera streams directly to local storage with no cloud upload, no subscriptions, and no monthly fees. Frigate performs real-time object detection (people, vehicles, animals) locally and integrates with Home Assistant for automation. Set up is straightforward: install Docker, configure your IP cameras' RTSP streams in Frigate's config file, and video records directly to attached storage. Your footage stays fully under your control.

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

## Installing Frigate

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

## Configuring Your Cameras

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

## Setting Up Object Detection

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

## Integrating with Home Assistant

For seamless integration, add the Frigate integration in Home Assistant by adding this to your `configuration.yaml`:

```yaml
frigate:
  url: "http://your-frigate-ip:5000"
  camera_ui_enabled: true
```

After restarting Home Assistant, you'll have access to Frigate camera entities, motion sensors, and can create automations based on detected objects.

## Storage and Retention

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

## Remote Access Without Cloud

To access your cameras remotely without cloud services, set up a VPN connection to your home network. WireGuard provides a lightweight, secure option that works well for remote camera viewing. Install WireGuard on your phone and home router, then connect to your network when you need to view live footage or review recordings.

Alternatively, Home Assistant's remote access through Nabu Casa works without opening ports on your router, though this does require a subscription. For completely local operation, WireGuard or a similar VPN solution is recommended.

## Performance Optimization

If you experience performance issues with multiple cameras, consider these adjustments:

- Enable hardware acceleration by adding `VAAPI` or `QSV` to your Docker runtime
- Reduce detection resolution for cameras you don't monitor closely
- Limit the number of detection zones and object types
- Use motion-based recording instead of continuous recording to save resources

## Conclusion

Frigate provides a powerful, private alternative to cloud-based security camera systems. Your footage stays on your hardware, costs are limited to upfront equipment purchases, and you have complete control over your data. Start with one or two cameras and expand as needed—the modular configuration makes adding new cameras straightforward.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
