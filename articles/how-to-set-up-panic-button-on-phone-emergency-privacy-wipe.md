---
layout: default
title: "How to Set Up Panic Button on Phone: Emergency Privacy Wipe"
description: "A practical guide for developers and power users to set up panic button emergency privacy wipe on Android and iOS. Includes Tasker, Shortcuts, and."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-panic-button-on-phone-emergency-privacy-wipe/
categories: [guides, security]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Setting up a panic button on your phone enables rapid data destruction when you face an emergency situation. Whether you're a journalist protecting sources, a developer handling sensitive credentials, or a power user prioritizing privacy, an emergency privacy wipe mechanism provides a critical safety net. This guide covers practical implementations for both Android and iOS, with automation scripts and code examples for developers.

## Understanding Emergency Privacy Wipe

A panic button triggers predefined actions that erase, hide, or secure sensitive data. Unlike standard factory resets, a well-configured privacy wipe selectively targets the data that matters most: credentials, contacts, messages, photos, and application data. The goal is speed and certainty—executing the wipe within seconds while minimizing the chance of partial execution.

The primary scenarios include:
- Forced device handover during border crossings or detentions
- Immediate threat of device compromise
- Physical assault or kidnapping situations
- Emergency evacuation with no time to manually secure devices

Modern mobile operating systems provide automation hooks that make panic button implementation accessible without specialized hardware.

## Android Implementation with Tasker

Tasker is the most flexible automation tool for Android. It responds to hardware buttons, NFC tags, and gesture triggers. The following profile executes a privacy wipe when you press the volume-down button three times within two seconds.

### Tasker Profile Configuration

Create a new profile in Tasker with the following trigger:
- **Event**: Hardware Button > Volume Down
- **Count**: 3
- **Within**: 2 seconds

### Privacy Wipe Task Actions

Configure the task with these actions in sequence:

```
1. Variable Set: %WIPE_ENABLED to "1"
2. Wait: 1 second
3. Variable Set: %CONFIRM to %WIPE_ENABLED
4. If: %CONFIRM = "1"
5.  Run Shell: cmd appops set --user 0 com.android.browser RESET_ALL
6.  Run Shell: pm clear com.android.browser
7.  Run Shell: pm clear com.android.providers.downloads
8.  SQL Delete: DELETE FROM content://sms/* (requires WRITE_SMS permission)
9.  Delete Directory: /sdcard/DCIM recursive
10. Delete Directory: /sdcard/Pictures recursive
11. HTTP Post: https://your-server.com/wipe flag="triggered"
12. Flash: Privacy wipe executed
13. End If
```

The shell commands require root access. For non-rooted devices, you can clear your own app's data and trigger remote wipe through device administrator APIs.

### Secure Wipe with Encryption

For developers building custom solutions, implement encrypted data containers that self-destruct:

```kotlin
// Android: EncryptedSharedPreferences with panic trigger
class SecureWipeManager(private val context: Context) {
    
    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()
    
    private val encryptedPrefs = EncryptedSharedPreferences.create(
        context,
        "secure_prefs",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )
    
    fun triggerPanicWipe() {
        // Clear encrypted preferences
        encryptedPrefs.edit().clear().apply()
        
        // Delete encryption keys
        val keyStore = KeyStore.getInstance("AndroidKeyStore")
        keyStore.load(null)
        keyStore.deleteEntry("privacy_wipe_key")
        
        // Clear app data programmatically (requires app owns this logic)
        val packageManager = context.packageManager
        val intent = packageManager.getLaunchIntentForPackage(context.packageName)
        intent?.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP)
        startActivity(intent)
        
        // Trigger remote wipe callback
        sendWipeConfirmation()
    }
    
    private fun sendWipeConfirmation() {
        val client = OkHttpClient()
        val request = Request.Builder()
            .url("https://your-server.com/api/wipe/confirm")
            .post(RequestBody.create(null, ByteArray(0)))
            .build()
        client.newCall(request).enqueue(object : Callback {
            override fun onFailure(call: Call, e: IOException) {}
            override fun onResponse(call: Call, response: Response) {}
        })
    }
}
```

## iOS Implementation with Shortcuts

iOS Shortcuts provides native automation without requiring a jailbroken device. While iOS restricts deep system access, you can still wipe app data, reset settings, and trigger remote lock.

### Creating the Shortcut

1. Open Shortcuts app
2. Create new shortcut named "Emergency Wipe"
3. Add actions:

```
- Wait: 0.5 seconds
- Show Alert: "Confirm Wipe" with "Wipe All Data" button
- Remove All Reminders
- Remove All Calendars
- Delete All Photos (requires confirmation)
- Clear Clipboard
- Run Shortcut: "Lockdown Mode" (separate shortcut)
- HTTP Request: POST to your-server.com/wipe
```

### Push Button Hardware Options

For hardware-triggered panic buttons on iOS:

- **AssistiveTouch**: Configure triple-click to trigger Shortcuts
- **Bluetooth button**: Pair a cheap Bluetooth remote and map it to Shortcuts
- **NFC tags**: Use NTAG213/215 tags placed on phone case or sticker

### iOS Device Wipe API

For MDM-managed devices, developers can trigger enterprise wipe:

```swift
// iOS: Managed API (requires MDM enrollment)
import ManagedSettings

class DeviceWipeManager {
    
    func triggerLocalWipe() {
        let store = ManagedSettingsStore()
        
        // This wipes all managed data
        store.wipeDeviceAndResetPasscode()
    }
    
    func triggerRemoteWipe() {
        // Requires MDM server integration
        let wipeRequest = WipeRequest(
            deviceUDID: UIDevice.current.identifierForVendor?.uuidString ?? "",
            wipeType: .enterprise,
            preserveActivationLock: false
        )
        
        MDMService.shared.submitWipeRequest(wipeRequest)
    }
}
```

## Network-Triggered Remote Wipe

The most reliable panic button operates independently of the device itself. Configure a server-side endpoint that triggers wipe on connected devices.

### Server Implementation

```python
# Python Flask server for remote wipe coordination
from flask import Flask, request, jsonify
from datetime import datetime, timedelta

app = Flask(__name__)

# Store device tokens and last check-in
devices = {}

@app.route('/api/register', methods=['POST'])
def register_device():
    data = request.json
    devices[data['device_id']] = {
        'token': data['push_token'],
        'last_seen': datetime.utcnow(),
        'wipe_pending': False
    }
    return jsonify({'status': 'registered'})

@app.route('/api/wipe', methods=['POST'])
def trigger_wipe():
    # Mark device for wipe - client polls this
    data = request.json
    device_id = data.get('device_id')
    
    if device_id in devices:
        devices[device_id]['wipe_pending'] = True
        devices[device_id]['wipe_triggered_at'] = datetime.utcnow()
        
        # Push notification to device (Firebase/APNs)
        send_push_wipe_notification(devices[device_id]['token'])
        
        return jsonify({'status': 'wipe_triggered'})
    
    return jsonify({'error': 'device_not_found'}), 404

@app.route('/api/check_wipe', methods=['GET'])
def check_wipe_status():
    device_id = request.args.get('device_id')
    
    if device_id in devices:
        device = devices[device_id]
        device['last_seen'] = datetime.utcnow()
        
        if device.get('wipe_pending'):
            return jsonify({'wipe': True})
    
    return jsonify({'wipe': False})
```

### Client Polling Script

```bash
#!/bin/bash
# Android/iOS polling script for remote wipe

DEVICE_ID="your-device-uuid"
SERVER_URL="https://your-server.com"

while true; do
    RESPONSE=$(curl -s "${SERVER_URL}/api/check_wipe?device_id=${DEVICE_ID}")
    
    if echo "$RESPONSE" | grep -q '"wipe":true'; then
        echo "Wipe triggered remotely - executing local wipe"
        
        # Android: Clear app data
        pm clear com.yourapp.package
        
        # iOS: Clear app via Shortcuts
        shortcuts run "Emergency Wipe"
        
        # Factory reset as final measure
        am start -a android.intent.action.MASTER_CLEAR
        
        break
    fi
    
    # Check every 10 seconds
    sleep 10
done
```

## Best Practices and Considerations

Test your panic button implementation thoroughly before relying on it in actual emergencies. False positives cause data loss; false negatives create dangerous overconfidence.

- **Test on secondary device**: Never test wipe logic on your primary device without backup
- **Verify wipe completion**: Ensure deleted data cannot be recovered with forensic tools
- **Maintain deniability**: Consider covering wipe actions with legitimate-looking error messages
- **Offline capability**: Network-triggered wipes fail without connectivity—always include local triggers
- **Multiple confirmation steps**: Prevent accidental activation while maintaining speed

For developers integrating these features into privacy-focused applications, consider the legal implications in your jurisdiction. Some countries require lawful access capabilities; others prohibit backdoors. Building responsible security tools means understanding both the threat model and the regulatory environment.

---


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
