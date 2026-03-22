---
layout: default
title: "Secure MQTT Broker Setup for IoT"
description: "Deploy a hardened Mosquitto MQTT broker with TLS, client certificate auth, and ACLs to keep your IoT network private and tamper-resistant"
date: 2026-03-22
author: theluckystrike
permalink: /secure-mqtt-broker-setup-iot/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

# Secure MQTT Broker Setup for IoT

MQTT is the messaging backbone of most smart home and IoT systems. Left unsecured, a public-facing MQTT broker leaks device state, sensor readings, and control commands to anyone who can connect. This guide sets up a hardened Mosquitto broker with TLS encryption, mutual certificate authentication, and topic-level access controls.

## Why Default MQTT Setups Are Dangerous

The default Mosquitto configuration listens on port 1883 with no authentication and no encryption. Anyone on the network — or the internet if port-forwarded — can:

- Subscribe to `#` and receive all messages from all devices
- Publish fake sensor readings or control commands
- Map your home's device topology from message patterns

This is not hypothetical. Shodan regularly indexes thousands of open MQTT brokers. Many broadcast door lock states, motion sensor triggers, and energy usage in real time.

## Prerequisites

- Ubuntu 22.04 or Debian 12 server
- Domain name or static IP for your broker
- Basic familiarity with OpenSSL

## Step 1 — Install Mosquitto

```bash
sudo apt update && sudo apt install -y mosquitto mosquitto-clients

# Stop the default instance — we'll configure before starting
sudo systemctl stop mosquitto
```

## Step 2 — Generate a Certificate Authority

You'll create your own CA to sign broker and client certificates. This enables mutual TLS — both sides prove their identity.

```bash
# Create directory structure
sudo mkdir -p /etc/mosquitto/certs
cd /etc/mosquitto/certs

# Generate CA key and certificate
sudo openssl genrsa -out ca.key 4096
sudo openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=IoT-CA/O=HomeNetwork/C=US"
```

## Step 3 — Generate the Broker Certificate

```bash
# Broker private key
sudo openssl genrsa -out broker.key 4096

# Certificate signing request
sudo openssl req -new -key broker.key -out broker.csr \
  -subj "/CN=mqtt.local/O=HomeNetwork/C=US"

# Sign the broker cert with your CA
sudo openssl x509 -req -days 825 -in broker.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out broker.crt

# Set strict permissions
sudo chmod 600 /etc/mosquitto/certs/*.key
sudo chown mosquitto:mosquitto /etc/mosquitto/certs/*
```

## Step 4 — Generate Client Certificates

Each IoT device gets its own certificate. This example creates one for a temperature sensor:

```bash
# Create a cert for each device
DEVICE="temp-sensor-01"

sudo openssl genrsa -out ${DEVICE}.key 4096
sudo openssl req -new -key ${DEVICE}.key -out ${DEVICE}.csr \
  -subj "/CN=${DEVICE}/O=HomeNetwork/C=US"
sudo openssl x509 -req -days 825 -in ${DEVICE}.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out ${DEVICE}.crt

# Copy key and cert to the device (via SCP or similar)
```

Repeat for each device: `thermostat-01`, `door-sensor-01`, `hub-01`, etc.

## Step 5 — Configure Mosquitto

Replace `/etc/mosquitto/mosquitto.conf` with a hardened configuration:

```ini
# /etc/mosquitto/mosquitto.conf

# Disable unencrypted listener
# listener 1883   <-- this line should NOT exist

# TLS listener only
listener 8883
protocol mqtt

# TLS certificates
cafile /etc/mosquitto/certs/ca.crt
certfile /etc/mosquitto/certs/broker.crt
keyfile /etc/mosquitto/certs/broker.key

# Require client certificates — no anonymous connections
require_certificate true
use_identity_as_username true

# ACL file for topic permissions
acl_file /etc/mosquitto/acl.conf

# Password file (used alongside cert auth for belt-and-suspenders)
password_file /etc/mosquitto/passwd

# Disable anonymous access
allow_anonymous false

# Persistence
persistence true
persistence_location /var/lib/mosquitto/

# Logging
log_dest file /var/log/mosquitto/mosquitto.log
log_type all

# Connection limits
max_connections 100
max_inflight_messages 20
```

## Step 6 — Configure Topic ACLs

The ACL file controls which clients can publish or subscribe to which topics:

```ini
# /etc/mosquitto/acl.conf

# temp-sensor-01 can only publish to its own topic
user temp-sensor-01
topic write home/sensors/temperature/01

# thermostat-01 can read temperature and write its own state
user thermostat-01
topic read home/sensors/temperature/#
topic write home/hvac/thermostat/01

# door-sensor-01 can only publish to door events
user door-sensor-01
topic write home/security/doors/01

# hub-01 has read access to everything
user hub-01
topic read #

# No user should write to system topics
# (implicit — no rules grant write to $SYS/#)
```

Topic ACLs follow the principle of least privilege: each device only touches what it needs.

## Step 7 — Create Password File (Belt-and-Suspenders)

Even with certificate auth, add a password as a second factor:

```bash
# Create password file with first user
sudo mosquitto_passwd -c /etc/mosquitto/passwd temp-sensor-01

# Add more users
sudo mosquitto_passwd /etc/mosquitto/passwd thermostat-01
sudo mosquitto_passwd /etc/mosquitto/passwd door-sensor-01
sudo mosquitto_passwd /etc/mosquitto/passwd hub-01

# Permissions
sudo chmod 640 /etc/mosquitto/passwd
sudo chown mosquitto:mosquitto /etc/mosquitto/passwd
```

## Step 8 — Start and Test

```bash
sudo systemctl enable --now mosquitto
sudo systemctl status mosquitto

# Test with mosquitto_sub using client cert
mosquitto_sub \
  --cafile /etc/mosquitto/certs/ca.crt \
  --cert /etc/mosquitto/certs/hub-01.crt \
  --key /etc/mosquitto/certs/hub-01.key \
  -h mqtt.local -p 8883 \
  -u hub-01 -P yourpassword \
  -t "home/#" -v

# Publish a test message from another terminal
mosquitto_pub \
  --cafile /etc/mosquitto/certs/ca.crt \
  --cert /etc/mosquitto/certs/temp-sensor-01.crt \
  --key /etc/mosquitto/certs/temp-sensor-01.key \
  -h mqtt.local -p 8883 \
  -u temp-sensor-01 -P yourpassword \
  -t "home/sensors/temperature/01" \
  -m '{"temp": 21.5, "unit": "C"}'
```

## Step 9 — Firewall Rules

Block all MQTT traffic except from trusted devices:

```bash
# Allow port 8883 only from LAN
sudo ufw allow from 192.168.1.0/24 to any port 8883
sudo ufw deny 8883

# Block legacy unencrypted port entirely
sudo ufw deny 1883
```

## Certificate Rotation

Client certificates expire. Automate rotation with a script:

```bash
#!/bin/bash
# Renew a device certificate
DEVICE=$1
DAYS=825
CA_DIR=/etc/mosquitto/certs

openssl genrsa -out ${CA_DIR}/${DEVICE}.key 4096
openssl req -new -key ${CA_DIR}/${DEVICE}.key \
  -out ${CA_DIR}/${DEVICE}.csr \
  -subj "/CN=${DEVICE}/O=HomeNetwork/C=US"
openssl x509 -req -days ${DAYS} \
  -in ${CA_DIR}/${DEVICE}.csr \
  -CA ${CA_DIR}/ca.crt \
  -CAkey ${CA_DIR}/ca.key \
  -CAcreateserial \
  -out ${CA_DIR}/${DEVICE}.crt

echo "Certificate renewed for ${DEVICE}. Deploy ${DEVICE}.key and ${DEVICE}.crt to device."
```

## Related Reading

- [How to Run Zigbee2MQTT Locally Without Vendor Cloud](/privacy-tools-guide/how-to-run-zigbee2mqtt-locally-for-smart-home-without-vendor/)
- [How to Set Up Falco for Container Runtime Security](/privacy-tools-guide/falco-container-runtime-security-setup/)
- [How to Configure iptables Rules from Scratch](/privacy-tools-guide/iptables-rules-from-scratch-guide/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
