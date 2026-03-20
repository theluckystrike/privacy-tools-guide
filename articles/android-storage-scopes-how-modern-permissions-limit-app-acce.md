---

layout: default
title: "Android Storage Scopes How Modern Permissions Limit App Acce"
description: "A developer guide to Android storage scopes, scoped storage, and how modern permissions control app access to files. Includes code examples and."
date: 2026-03-15
author: "Privacy Tools Guide"
permalink: /android-storage-scopes-how-modern-permissions-limit-app-acce/
categories: [guides]
voice-checked: true
reviewed: true
score: 8
intent-checked: true
tags: [privacy-tools-guide]
---

{% raw %}

Scoped storage (introduced in Android 10 and mandatory since Android 12) limits apps to their own sandbox directories and prevents broad read-write access to all files on external storage. Instead of requesting READ_EXTERNAL_STORAGE permission that granted access to all files, apps now use the MediaStore API for shared media or request user file picks through the Storage Access Framework. This fundamentally reduces privacy risks by preventing apps from accessing your documents, photos, or other sensitive files without explicit per-file user consent.

## The Evolution of Android Storage Permissions

Before Android 10, apps could request broad storage permissions that granted read and write access to nearly all files on external storage. The `READ_EXTERNAL_STORAGE` and `WRITE_EXTERNAL_STORAGE` permissions provided nearly unrestricted access to the device's shared storage, including other apps' data directories. This open model created significant privacy risks—apps could potentially read sensitive documents, photos, and downloaded files without explicit user awareness.

Google introduced scoped storage in Android 10 as a privacy-first approach, gradually making it mandatory in later versions. Under this model, each app receives an isolated sandbox containing its own files, while access to shared storage becomes mediated through specific permissions and user-granted file picks.

## Understanding Scoped Storage Architecture

Scoped storage divides storage into distinct categories with different access rules:

**App-specific storage** provides a private directory for each app, accessible without any permissions. Files stored using `context.getExternalFilesDir()` or `context.filesDir` remain isolated to your application and get deleted when the user uninstalls the app.

**Shared storage** encompasses media files (images, videos, audio), documents, and downloads accessible to multiple apps. Access requires different permissions depending on the Android version and file type.

**Media Store API** allows querying and modifying media files (images, videos, audio) without requesting storage permissions on Android 13+. The system acts as an intermediary, presenting only files the app has legitimately created or that users have explicitly selected.

## Storage Permissions by Android Version

The permission model varies significantly across Android versions:

### Android 9 and Below
Apps could request `WRITE_EXTERNAL_STORAGE`, which implicitly granted `READ_EXTERNAL_STORAGE`. These permissions provided access to the entire shared storage hierarchy.

```xml
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
```

### Android 10 (API 29)
Scoped storage became available as an opt-in feature. Developers could set `android:requestLegacyExternalStorage="true"` in the manifest to revert to legacy storage behavior temporarily. Google Play started requiring justified use cases for broad storage access.

### Android 11 (API 30)
Scoped storage became mandatory for apps targeting API 30+. The `MANAGE_EXTERNAL_STORAGE` permission replaced broad access but requires users to manually grant permission through a settings screen—Google restricts this permission to specific use cases like file managers.

### Android 12+ (API 31+)
Additional granular permissions emerged. `READ_EXTERNAL_STORAGE` requires `android:maxSdkVersion="32"` for automatic granting on Android 13+. Android 13 introduced photo and video picker, eliminating the need for storage permissions in many scenarios.

## Practical Code Examples

### Accessing App-Specific Files

App-specific storage requires no permissions and provides secure, isolated file storage:

```kotlin
// Get app-specific external storage directory
val appDir = context.getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS)
val myFile = File(appDir, "my-document.pdf")

// Write without any permissions
FileOutputStream(myFile).use { output ->
    output.write(fileContent)
}
```

### Using Media Store API (Android 10+)

Querying existing media files without requesting storage permissions:

```kotlin
// Android 13+ - no permission needed for reading
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    val projection = arrayOf(
        MediaStore.Images.Media.DISPLAY_NAME,
        MediaStore.Images.Media.DATE_ADDED
    )
    
    val cursor = contentResolver.query(
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
        projection,
        null,
        null,
        "${MediaStore.Images.Media.DATE_ADDED} DESC"
    )
    
    cursor?.use {
        while (it.moveToNext()) {
            val name = it.getString(0)
            val date = it.getLong(1)
            // Process image metadata
        }
    }
}
```

### Requesting Photo Access (Android 13+)

The photo picker provides the most privacy-conscious approach:

```kotlin
// Launch system photo picker - no permission request needed
val pickMedia = registerForActivityResult(ActivityResultContracts.PickVisualMedia()) { uri ->
    uri?.let {
        // User selected photos, grant temporary access
        contentResolver.takePersistableUriPermission(
            it,
            Intent.FLAG_GRANT_READ_URI_PERMISSION
        )
        // Process the selected image
    }
}

// Select multiple images
pickMedia.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly()))
```

### Legacy Storage Access (File Managers Only)

Apps genuinely requiring broad file access must declare `MANAGE_EXTERNAL_STORAGE` and guide users to enable it in settings:

```xml
<uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE" />
```

```kotlin
if (!Environment.isExternalStorageManager()) {
    val intent = Intent(Settings.ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION)
    intent.data = Uri.parse("package:$packageName")
    startActivity(intent)
}
```

Google typically rejects apps using this permission unless they function as file managers or have documented, essential file-handling requirements.

## Best Practices for Developers

**Default to app-specific storage** when your app creates files. This approach requires no permissions, provides automatic cleanup on uninstall, and faces no restrictions.

**Use Media Store API** for accessing shared media. On modern Android versions, you can read media files without permissions while the system mediates write access through user prompts.

**Implement photo picker** for selecting user media. This approach provides the best user experience, requires no permissions, and clearly communicates what data your app accesses.

**Test across Android versions** using the `maxSdkVersion` attribute in permission declarations to ensure smooth behavior across the Android ecosystem.

## Implications for Users

Modern storage permissions give users more control over their data. Users can see which apps have storage permissions through system settings, and Android prompts for permission just-in-time rather than at install. The photo picker means users share specific images intentionally rather than granting blanket photo access.

Reviewing app permissions regularly and understanding that "storage access" no longer means "access everything" helps users maintain better privacy hygiene on their devices.

---


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
